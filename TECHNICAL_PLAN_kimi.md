# TECHNICAL PLAN — Open Video Discovery for PeerTube

> Consolidated document, 2026-09-25. Merges the initial architecture plan with three rounds
> of adversarial review and fact-checking against the live PeerTube ecosystem.
> Evidence-based corrections are resolved inline. Open empirical questions are marked **[VERIFY W1]**.

---

## 0. Strategic Positioning (read this first)

This is not a crawler project or a search engine project. The first community crawler
(**PeerTube Index**) was retired precisely when official search arrived; the project's lead
stated "I will stop maintaining PeerTube Index" once Sepia Search launched. **A discovery
site lives or dies on its curation policy, not its crawler.** The real product is a
*transparent, appealable, rule-based moderation process combined with chronological-only
ordering*.

### Competitive landscape (verified 2026-09)

| Project | What it is | Stack | What it proves for us |
|---|---|---|---|
| **Sepia Search** (official) | Federation-wide video search | Meilisearch; index built on Elasticsearch pre-launch (2020), migrated before release | Uses the moderated instance registry; defines an open search-index API others (e.g. searxng) consume |
| **PeerSeek** | Browse/discovery index: ~940K videos from ~1,600 instances | Small Go services, PostgreSQL + Valency, Kubernetes | **PostgreSQL FTS is fast enough at near-full public federation scale** ("deliberately no OpenSearch or Elasticsearch"); no user accounts — anonymous `profile_token` in localStorage; hybrid newest-page + deep-pagination crawl; tag-backfill pass; dead-instance retention job; robots.txt + descriptive UA compliance |
| **Fedi.Video** | Human-curated, manually checked index; companion Mastodon bot | — | **Occupies the manifesto's niche** (calm, educational, safe). Cannot match us on transparent rules, public appeals, or scale |
| **PeerTube Browser** | ANN similarity recommendations | ANN index, AGPL-3.0 | Engagement-style ranking exists; we explicitly do not compete here |

**Differentiators (none of the above do all four):**
1. Chronological + manual ordering only — no engagement ranking anywhere in the stack
2. Public moderation log + appeals queue with SLA (anti-manipulation rules published)
3. Creator-submitted *and* crawled, with transparent inclusion rules
4. Sepia-compatible search API exposed, so the index survives the site

**Pre-build requirements (empirical, cannot be resolved by more review):**
- [ ] Probe 5 live instances' real API behavior (rate limits, `include` bitmask, `Retry-After`)
- [ ] Talk to 2–3 instance admins and the Fedi.Video curator about crawl/embed policy
- [ ] Write the public positioning/comparison page — including the feed anti-manipulation policy (§9)

---

## 1. Core Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Video hosting | None — PeerTube instances host; we are a pass-through | Manifesto; creators keep control |
| Stored data | Metadata only + proxied thumbnails (§10) | Minimal legal/ethical surface |
| Ordering | Chronological + manual curation, **plus public per-channel feed rate limits** (§9) | Chronology alone is maximally spam-exploitable; limits are a published anti-manipulation policy, not an engagement algorithm |
| Search backend | **PostgreSQL FTS only. No Meilisearch, no Elasticsearch** — PeerSeek proved it to ~940K videos | Simplicity, ops burden, ethos |
| Personalization | Anonymous `profile_token` in localStorage; **no user accounts** | PeerSeek-validated; no passwords, no GDPR login surface, minimal data collection. Trade-off documented: clearing site data loses follows; no cross-device sync (accepted) |
| Crawl model | Hybrid: creator submissions **and** seeded instance registry | Registry seed: `https://instances.joinpeertube.org/api/v1/instances` (canonical moderated list; consumed by Sepia and PeerSeek) |
| Federation API (phase 2) | Sepia-compatible search API shape; optional ActivityPub endpoint for a followable chronological stream | Ecosystem interoperability; manifesto "if this site fails, nothing is lost" |
| Container platform | Docker Compose on one VPS. **No Kubernetes** until ~500K videos | PeerSeek needs K8s at federation scale; we won't for a long time |

---

## 2. Architecture

```
Creator submits URL ──► Submission API ──► Validator ──► Celery: crawl:new (high priority)
Registry sync (joinpeertube.org) ────────► Instance registry ──► Celery: crawl:deep (low)
                                                        │
                              Celery Beat (scheduler) ──┤
                                                        ▼
                                   Per-domain adaptive token bucket (Redis)
                                                        │
                                                        ▼
                                              PeerTube Instance REST API
                                                        │
                                                        ▼
                                    PostgreSQL (metadata + tsvector search index)
                                     Redis (queues, rate-limit state, cache)
                                                        │
              ┌─────────────────────────────────────────┼──────────────────────────┐
              ▼                                         ▼                          ▼
       Public Search/Browse API                 Thumbnail proxy/cache        Admin panel
       (Sepia-compatible shape)                 (short TTL, opt-out)         (moderation, appeals)
              │
              ▼
        Next.js frontend (per-page rendering strategy, §11)
```

**Components:** submission service · crawler worker pool · search service · thumbnail proxy ·
admin/appeals panel · transparency page (public moderation log, rendered from the DB).

---

## 3. Data Model (PostgreSQL)

```sql
instances        (id, domain, name, version, last_crawl_at, last_seen_at,
                  is_blacklisted, robots_allowed, thumbnail_opt_out)
channels         (id, instance_id, handle, display_name, description, avatar_url,
                  actor_id,            -- kept ActivityPub-shaped for phase 2
                  sync_tier,           -- hot | active | cold (§7.4)
                  submitted_at, last_synced_at, is_active)
videos           (id, channel_id, uuid, peertube_video_id,
                  canonical_video_id,  -- FK to canonical copy for cross-instance dedup
                  title, description, tags,
                  nsfw_flags  JSONB,   -- {"violent": bool, "shocking": bool, "sexually_explicit": bool} (PeerTube ≥ v7.2)
                  nsfw_policy TEXT DEFAULT 'warn',  -- display|warn|blur|hide; site default 'warn'
                  category_id, language, duration,
                  thumbnail_url,       -- origin URL (we proxy, §10)
                  watch_url,           -- reconstructed https://{domain}/w/{shortUUID} — NOT the submitted URL
                  published_at, last_synced_at, is_active)
video_mirrors    (video_id, instance_id, watch_url, last_seen_at)   -- "Also available on…"
categories       (id, name, slug)          -- PeerTube native 15-category taxonomy + admin collections
submissions      (id, submitted_url, type, status, submitter_hash, created_at)  -- rate-limited by IP hash; no accounts
profiles         (token UUID PK, created_at)                            -- anonymous follows
profile_follows  (token, channel_id, created_at)
blocklist        (subject_type, subject_id, source, reason, appeal_status, public_log)
                 -- source: 'local' | 'remote-mute-list:{url}'; public_log renders on transparency page
```

**Search index — built correctly from day one:**

```sql
ALTER TABLE videos ADD COLUMN search_vector tsvector;
CREATE TRIGGER videos_search_vector_update BEFORE INSERT OR UPDATE ON videos
  FOR EACH ROW EXECUTE FUNCTION
  tsvector_update_trigger(search_vector, 'pg_catalog.english', title, description, tags);

CREATE INDEX videos_search_idx   ON videos USING GIN(search_vector);   -- FTS filtering
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX videos_title_trgm   ON videos USING GiST(title gist_trgm_ops(siglen=512)); -- top-N fuzzy ranking
```

Do **not** compute `to_tsvector()` per query. GiST (not GIN) on `title` because it's a
top-N ranking index; GIN on `search_vector` for match filtering.

**URL canonicalization:** normalize all PeerTube URL variants —
`/w/{shortUUID}` (22-char), `/videos/watch/{uuid}` (36-char), `/videos/embed/{uuid}`,
`/c/{channel}`, `/video-channels/{name}` — to `(domain, uuid)`. Store the reconstructed
watch URL, never the submitted string.

---

## 4. PeerTube API Integration

**Endpoints (public reads, no auth):**
- Video: `GET /api/v1/videos/{id}` — title, description, thumbnail path, tags, category, duration, publishedAt, NSFW flags
- Channel: `GET /api/v1/video-channels/{name}` — metadata + `videosCount` (cheap liveness check)
- Channel videos: `GET /api/v1/video-channels/{name}/videos`
- Instance health: `GET /api/v1/server/stats` **[VERIFY W1 — confirm exact endpoint on target instances]**

**Pagination:** offset-based — `start=0&count=100`, then `start=100&count=100`; `count` max ~100.
Sort with `sort=-publishedAt` for chronological crawls.

**Field bitmask:** `include` is a bitmask; `include=1` for cheap list passes (sparse tags),
`include=63` for full sync. **[VERIFY W1 against live instances; PeerTube uses an include
bitmask but confirm values per API version]**

**Rate limits:** PeerTube's documented default is ~50 requests / 10s per IP, but it is
instance-configurable and frequently lowered. **Never hardcode.** Adaptive per-domain token
bucket (§7.2), honor `Retry-After` (seconds and HTTP-date formats), back off on 429/5xx.

**Respect:** robots.txt checked per instance before any crawl; crawl halts instantly when
disallowed. Descriptive UA: `OpenVideoDiscoveryBot/1.0 (+https://yourdomain.org/bot)` with a
published bot-policy page and an opt-out mechanism honored within one sync cycle.

**Embed note:** PeerTube allows uploaders to restrict which domains may embed. This
validates no-embed-by-default — and even if we wanted embeds, many creators block them.

---

## 5. Crawler Design (Celery)

### 5.1 Queues

| Queue | Priority | Workers | Use |
|---|---|---|---|
| `crawl:new` | High | 4 | New submissions, user-triggered |
| `crawl:sync` | Normal | 2 | Scheduled channel syncs |
| `crawl:liveness` | Low | 1 | Cheap 404/`videosCount` checks |
| `crawl:deep` | Lowest | 1 | Registry-wide back-catalog discovery |

Separate queues prevent a deep instance crawl from blocking a user's submission.

### 5.2 Per-domain adaptive rate limiter

Celery's `rate_limit` is per-task, not per-domain — implement a Redis token bucket:

```python
def rate_limited_get(domain, url):
    key = f"rl:{domain}"
    min_interval = float(redis.get(f"{key}:interval") or 2.0)  # start conservative: 0.5 req/s
    last = float(redis.get(f"{key}:last") or 0)
    wait = min_interval - (now() - last)
    if wait > 0: sleep(wait)
    r = requests.get(url, headers=UA, timeout=15)
    if r.status_code == 429:
        redis.set(f"{key}:interval", min_interval * 2)          # halve speed
        retry in parse_retry_after(r)                            # seconds + HTTP-date
    elif r.ok and min_interval > 0.5:
        redis.set(f"{key}:interval", max(0.5, min_interval * 0.9)) # ease upward
    redis.set(f"{key}:last", now())
    return r
```

Failed jobs go to a **dead-letter queue** with exponential backoff (the plan's single
silent-failure point otherwise).

### 5.3 Hybrid pagination (PeerSeek-proven)

Always re-check the **newest page** of each followed channel/instance (fresh uploads appear
quickly), while separately resuming **deep pagination** for back-catalogs not yet fully
indexed. Track per-channel `deep_cursor` so interrupted deep crawls resume.

### 5.4 Tag-backfill pass

List endpoints return sparse tags. A separate job fetches `include=63` details for tagless
videos and records progress so the same videos aren't retried forever.

### 5.5 Sync tiers (Celery Beat)

| Tier | Criterion | Interval |
|---|---|---|
| Hot | >10 followers or >3 submissions in 7 days | 2–4 h |
| Active | ≥1 follower or submitted in last 30 days | 6–12 h |
| Cold | no activity in 30 days | 72 h, liveness check first |

### 5.6 Retention

Channel 404 or instance dark → mark `is_active=false` immediately (watch links still
explain status). **Purge** videos whose instance hasn't been seen in N days (~30) so the
index reflects what is actually watchable. Mirrors in `video_mirrors` keep "also available
on" links alive across instance outages.

---

## 6. Deduplication

PeerTube has no built-in duplicate detection. On insert, match candidate canonical videos
by `(title, duration ±5s, published_at ±1h)` across instances; link via
`canonical_video_id` and record alternates in `video_mirrors`. Surface "Also available on:
[instance list]" on video cards — genuinely useful to users behind flaky instances.

---

## 7. Search Service

- Postgres FTS only. `WHERE search_vector @@ plainto_tsquery('english', $1) ORDER BY ts_rank(...)`, plus GiST `similarity(title, $1)` for fuzzy top-N.
- Filters: category, language, duration band, instance, NSFW policy.
- **Meilisearch/Elasticsearch: removed from the roadmap.** Revisit only on measured evidence (search-as-you-type >200 ms at p95, zero-result typo complaints, CJK tokenization demand). PeerSeek's ~940K-video corpus is near the entire *accessible public* federation **[don't claim "entire federation" publicly — ~1,600 of ~1,689 listed instances, many private/tiny/dead]**.
- Phase 2: expose a **Sepia-compatible search API** so other tools (searxng, etc.) can consume our index.

---

## 8. NSFW Handling (PeerTube ≥ v7.2 taxonomy)

NSFW is no longer binary. Store `nsfw_flags` JSONB (`violent`, `shocking`,
`sexually_explicit`). Site default per-flag policy: **warn** (never silent-hide — the
manifesto's transparency ethos), overridable per session via cookie: display / warn / blur /
hide.

---

## 9. Moderation, Anti-Abuse & Appeals (the actual product)

PeerTube 8.3 (2026-09) added **remote mute list** and **word list** subscriptions. Use them.

1. **Subscribe** to well-known community remote mute lists from day one; auto-import with source attribution in `blocklist.source`.
2. **Word lists** flag new submissions for a human review queue — humans only see flagged items, not the whole firehose.
3. **Local blocklist** for our own decisions, in the same format so PeerTube admins can re-subscribe to ours (builds goodwill, reuses ecosystem tooling).
4. **Appeals:** every block entry has `appeal_status`; appeals logged (anonymized) on the public transparency page; response SLA 7 days.
5. **Spam economics — the part chronology-only sites get wrong:** a channel publishing 20 videos/day floods every chronological feed. Policy: **max 3 videos per channel per day appear in default/category feeds; overflow lives on the channel page.** Published, uniform, appealable. Frame it on the site as an *anti-manipulation* rule — because chronology with no injection cap is not neutral, it's exploitable.
6. **Submission hygiene:** per-IP-hash rate limit (10/day), CAPTCHA, dedup by canonical identity.
7. **Takedown:** simple form + email; action within 48 h; soft-delete (`is_active=false`) for audit, never hard-delete.

---

## 10. Thumbnails — proxy and cache, do not hotlink

PeerTube generates/serves thumbnails on demand; hundreds of concurrent hotlinked thumbnail
hits from our browse pages land on 2-user instances — the opposite of protecting small
creators. Instead: **proxy thumbnails through our cache with short TTLs** (legally
low-risk; standard search-engine practice), honor per-instance `thumbnail_opt_out`
instantly. Cheaper for us, kinder to small instances, and dead instances' thumbs keep
rendering. (Reverses the earlier "hotlink by default" idea — that default put the burden on
the least able to complain.)

---

## 11. Frontend (Next.js App Router)

**Rendering strategy per page type:**

| Page | Strategy | Why |
|---|---|---|
| `/` home, category feeds | ISR 60 s | Frequent, not per-request |
| `/search` | SSR / client fetch | Query-dependent, no SEO value in results |
| `/channel/{handle}@{domain}` | ISR 300 s | Mostly static |
| `/submit`, `/about`, `/transparency` | SSG | Static |
| Video detail | ISR | Fresh metadata, structured data |

**SEO decision (resolved — the rounds found a contradiction):** SSR-for-SEO investment
collides with cross-domain canonicals (Google treats our page as duplicate and drops it —
you cannot simultaneously rank and defer). **Ship v1 with Option A (manifesto-pure):** video
detail pages exist for UX; `rel=canonical` points to the origin watch URL; growth is
Fediverse word-of-mouth, not Google. **Option B (self-canonical + JSON-LD VideoObject +
prominent origin links) is a config flag** flipped only by an explicit decision if organic
search becomes a stated goal. Do not build half of each.

**UX rules:** every video card links out to the creator's server; no pulled-back view/like
counts anywhere (the UI literally cannot rank by them); optional embed preview respecting
uploader embed restrictions; sitemap.ts / robots.ts; NSFW warn overlays per §8.

---

## 12. Personalization (no accounts)

- Browser generates random `profile_token`, stored in `localStorage`; follows and per-session preferences attach to it.
- "My feed" = chronological merge of followed channels.
- Clearing site data = new anonymous profile (follows lost). Accepted, documented, and arguably on-brand: we hold nothing.
- Phase 2 (optional): ActivityPub endpoint so the chronological feed itself is followable from Mastodon — the natural fit for a no-algorithm stream.

---

## 13. Deployment, Cost, Operations

| Scale | RAM | ~Cost/mo |
|---|---|---|
| < 10K videos | 4 GB | ~$20 |
| 10K–100K | 8 GB | ~$40 |
| 100K–500K | 16 GB | ~$80 |
| 500K+ | 32 GB | ~$160+ |

Docker Compose: PostgreSQL, Redis, Celery workers + Beat, Next.js, thumbnail proxy
(Caddy/nginx). Backups: nightly `pg_dump`, retain 7 daily + 4 weekly, off-site object storage.

**Monitoring:** Uptime Kuma + Sentry + crawl metrics:
- per-instance 4xx/5xx rate (sustained 403 → auto-backoff + review flag)
- `crawl:new` queue depth alert at 100 (submission backlog)
- sync staleness: any channel >2× its tier interval
- 429 hit rate; adjust concurrency when high

---

## 14. Milestones

**Honest scope note:** the original 6-week MVP silently absorbed a moderation surface
(appeals SLA, transparency page, mute-list subscriptions, feed caps) that exceeds that
budget. The plan's differentiation *is* the moderation surface — cutting it launches a
worse PeerSeek. **Realistic solo timeline: 10–12 weeks.**

- **W1** — Schema (incl. tsvector trigger, NSFW taxonomy), URL parser, submission endpoint. **[W1 verification battery: rate limits, include bitmask, server/stats endpoint, Retry-After on 5 live instances]**
- **W2** — Celery queues + Beat + Redis token-bucket limiter; single-video crawler; instance registry seeded from joinpeertube.org; robots.txt gate + UA
- **W3** — Channel crawler, hybrid pagination, deep cursors, dead-letter queue
- **W4** — Three-tier sync; liveness + retention jobs; tag backfill
- **W5** — Search API (GIN/GiST); Next.js browse + channel + video pages (ISR)
- **W6** — NSFW warn overlays; remote mute-list subscription; word-list flagging; review queue
- **W7** — Thumbnail proxy + opt-out; feed per-channel caps
- **W8** — Profile tokens + follows + "my feed"
- **W9** — Transparency page, appeals queue, bot-policy page, positioning/comparison page
- **W10** — Seed ~20 known educational instances; hardening; launch. (W11–12 buffer)
- **Phase 2** — Sepia-compatible search API; optional ActivityPub endpoint; multilingual UI

---

## 15. Risk Register

| Risk | Mitigation |
|---|---|
| Differentiation failure vs Sepia / PeerSeek / Fedi.Video | Lead with transparent appealable rules + chronological-only ordering; publish comparison page **before writing crawler code** |
| Curation burden kills the project (PeerTube Index precedent) | Remote mute lists + word lists from day one; humans review flagged items only |
| Chronological feed spam-flooding | Public per-channel feed caps (§9.5) — framed as anti-manipulation policy |
| Instance hostility (429s, robots blocks) | Descriptive UA, robots compliance, adaptive token bucket, public bot policy |
| Stale index | `is_active` flags + 30-day dark-instance purge + mirrors |
| Thumbnail load on small instances | Proxy/cache with instant opt-out (§10) |
| SEO self-contradiction | Option A canonical-to-origin shipped deliberately; Option B behind explicit flag (§11) |
| Legal | Metadata-only + proxied thumbs; 48 h takedown; soft-delete audit trail |
| Timeline underestimation | 10–12-week honest budget; moderation surface is non-negotiable scope |

---

## 16. Definition of Done (v1)

- [ ] Submit a video URL or a channel; metadata appears without human approval
- [ ] Search returns ranked results <200 ms p95 at 50K videos
- [ ] Default feeds are chronological with published per-channel caps
- [ ] Every moderation action appears on the public transparency page with reason + appeal status
- [ ] Appeals answered within 7 days
- [ ] robots.txt and opt-out honored; descriptive UA published
- [ ] No engagement metric influences any ordering anywhere
- [ ] All code open; index exportable; Sepia-compatible API spec'd (implementation phase 2)
