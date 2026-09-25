# TECHNICAL PLAN — Open Video Discovery for PeerTube

> Consolidated 2026-09-25. This supersedes `TECHNICAL_PLAN_Deepseek.md` and
> `TECHNICAL_PLAN_kimi.md`, which are kept in `/docs/archive/` for history only.
> Corrections from three rounds of review against the live PeerTube ecosystem are
> resolved inline. Open empirical questions are marked **[VERIFY W1]**.

## 0. Strategic Positioning (read this first)

This is not a crawler project or a search-engine project. PeerTube's first community
crawler, **PeerTube Index**, was retired the moment official search (Sepia Search)
launched — its maintainer said outright he'd stop maintaining it once Sepia shipped.
**A discovery site lives or dies on its curation policy, not its crawler.** The real
product here is a *transparent, appealable, rule-based moderation process combined
with chronological-only ordering* — something none of the existing tools do.

### Competitive landscape

| Project | What it is | Stack | What it proves for us |
|---|---|---|---|
| **Sepia Search** (official) | Federation-wide search | Meilisearch | Defines the ecosystem's open search-index API shape |
| **PeerSeek** | Browse/discovery, ~940K videos, ~1,600 instances | PostgreSQL + Valency, K8s | **Postgres FTS is fast enough at near-full federation scale.** No accounts — anonymous `profile_token`. Hybrid crawl (newest-page + deep pagination). Tag-backfill pass. Dead-instance retention. |
| **Fedi.Video** | Human-curated index + Mastodon bot | — | Occupies our manifesto's niche today. Can't match us on published rules, public appeals, or scale. |
| **PeerTube Browser** | ANN similarity recommendations | ANN index | Engagement-style ranking already exists elsewhere — we deliberately don't compete here |

**Four differentiators, and no existing tool has all four:**
1. Chronological + manual ordering only — no engagement ranking anywhere in the stack.
2. Public moderation log + appeals queue with a published SLA.
3. Creator-submitted *and* crawled, with transparent, uniform inclusion rules.
4. A Sepia-compatible search API exposed in phase 2, so the index survives even if the site doesn't.

**Before writing any crawler code:**
- [ ] Probe 5 live instances for real API behavior (rate limits, `include` bitmask, `Retry-After` format)
- [ ] Talk to 2–3 instance admins and the Fedi.Video curator about crawl/embed etiquette
- [ ] Publish the positioning/comparison page — including the feed anti-manipulation policy (§9) — before launch

---

## 1. Core Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Video hosting | None — pass-through only | Manifesto; creators keep full control |
| Stored data | Metadata only + proxied thumbnails (§10) | Minimal legal/ethical surface |
| Ordering | Chronological + manual curation, **plus public per-channel feed rate limits** (§9) | Pure chronology is maximally spam-exploitable; a published, uniform cap is a policy, not an algorithm |
| Search backend | **PostgreSQL FTS only** — no Meilisearch, no Elasticsearch | PeerSeek proved this scales to ~940K videos |
| Personalization | Anonymous `profile_token` in `localStorage`, no accounts | No passwords, no login GDPR surface, minimal data collected |
| Crawl model | Hybrid: creator submissions + seeded instance registry | Registry seed: `instances.joinpeertube.org` API (canonical moderated list, also consumed by Sepia/PeerSeek) |
| Federation (phase 2) | Sepia-compatible search API; optional ActivityPub endpoint | Ecosystem interoperability; "if this site fails, nothing is lost" |
| Deploy | Docker Compose, one VPS. No Kubernetes until ~500K videos | PeerSeek needs K8s at full federation scale; we won't for a long time |

---

## 2. Architecture

```
Creator submits URL ──► Submission API ──► Validator ──► Celery: crawl:new (high)
Registry sync (joinpeertube.org) ──────────► Instance registry ──► Celery: crawl:deep (low)
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
             ┌────────────────────────────────────────────┼───────────────────────────┐
             ▼                                             ▼                           ▼
      Public Search/Browse API                     Thumbnail proxy/cache        Admin panel
      (Sepia-compatible shape)                      (short TTL, opt-out)     (moderation, appeals)
             │
             ▼
      Next.js frontend (per-page rendering strategy, §11)
```

**Components:** submission service · crawler worker pool · search service · thumbnail proxy · admin/appeals panel · transparency page (rendered from the DB, not hand-maintained).

---

## 3. Data Model (PostgreSQL)

```sql
instances        (id, domain, name, version, last_crawl_at, last_seen_at,
                  is_blacklisted, robots_allowed, thumbnail_opt_out)
channels         (id, instance_id, handle, display_name, description, avatar_url,
                  actor_id,            -- kept ActivityPub-shaped for phase 2
                  sync_tier,           -- hot | active | cold (§5.5)
                  submitted_at, last_synced_at, is_active)
videos           (id, channel_id, uuid, peertube_video_id,
                  canonical_video_id,  -- FK to canonical copy, cross-instance dedup
                  title, description, tags,
                  nsfw_flags  JSONB,   -- {"violent":bool,"shocking":bool,"sexually_explicit":bool} (PeerTube ≥ v7.2)
                  nsfw_policy TEXT DEFAULT 'warn',   -- display|warn|blur|hide
                  category_id, language, duration,
                  thumbnail_url,       -- origin URL (we proxy, §10)
                  watch_url,           -- reconstructed https://{domain}/w/{shortUUID} — never the submitted string
                  published_at, last_synced_at, is_active)
video_mirrors    (video_id, instance_id, watch_url, last_seen_at)   -- "Also available on…"
categories       (id, name, slug)      -- PeerTube's native 15 categories + admin collections
submissions      (id, submitted_url, type, status, submitter_hash, created_at)
profiles         (token UUID PK, created_at)
profile_follows  (token, channel_id, created_at)
blocklist        (subject_type, subject_id, source, reason, appeal_status, public_log)
                 -- source: 'local' | 'remote-mute-list:{url}'
```

**Search index:**

```sql
ALTER TABLE videos ADD COLUMN search_vector tsvector;
CREATE TRIGGER videos_search_vector_update BEFORE INSERT OR UPDATE ON videos
  FOR EACH ROW EXECUTE FUNCTION
  tsvector_update_trigger(search_vector, 'pg_catalog.english', title, description, tags);

CREATE INDEX videos_search_idx ON videos USING GIN(search_vector);
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX videos_title_trgm ON videos USING GiST(title gist_trgm_ops(siglen=512));
```

Never compute `to_tsvector()` per query. GiST on `title` because it's a top-N ranking index; GIN on `search_vector` for match filtering.

**URL canonicalization:** normalize every PeerTube URL variant (`/w/{shortUUID}`, `/videos/watch/{uuid}`, `/videos/embed/{uuid}`, `/c/{channel}`, `/video-channels/{name}`) to `(domain, uuid)`. Always store the reconstructed watch URL — never the raw submitted string.

---

## 4. PeerTube API Integration

- **Video:** `GET /api/v1/videos/{id}` → title, description, thumbnail path, tags, category, duration, `publishedAt`, NSFW flags
- **Channel:** `GET /api/v1/video-channels/{name}` → metadata + `videosCount` (cheap liveness check)
- **Channel videos:** `GET /api/v1/video-channels/{name}/videos`, paginated
- **Instance health:** `GET /api/v1/server/stats` **[VERIFY W1 on target instances]**

**Pagination:** offset-based, `start`/`count`, max ~100/page. `sort=-publishedAt` for chronological crawls.

**Field bitmask:** `include=1` for cheap list passes, `include=63` for full sync. **[VERIFY W1 — confirm bitmask values against live API version]**

**Rate limits:** default is ~50 req/10s per IP, but instance-configurable — never hardcode. Adaptive per-domain token bucket, honor `Retry-After` (both seconds and HTTP-date formats), back off on 429/5xx.

**Respect:** check `robots.txt` per instance before any crawl; halt instantly if disallowed. Descriptive UA: `OpenVideoDiscoveryBot/1.0 (+https://yourdomain.org/bot)`, published bot-policy page, opt-out honored within one sync cycle.

**Embeds:** PeerTube lets uploaders restrict which domains can embed their videos — this reinforces the no-embed-by-default UX decision (§11).

---

## 5. Crawler Design (Celery)

### 5.1 Queues

| Queue | Priority | Workers | Use |
|---|---|---|---|
| `crawl:new` | High | 4 | New submissions, user-triggered |
| `crawl:sync` | Normal | 2 | Scheduled channel syncs |
| `crawl:liveness` | Low | 1 | Cheap 404 / `videosCount` checks |
| `crawl:deep` | Lowest | 1 | Registry-wide back-catalog discovery |

### 5.2 Per-domain adaptive rate limiter

Celery's `rate_limit` is per-task, not per-domain — implement a Redis token bucket instead:

```python
def rate_limited_get(domain, url):
    key = f"rl:{domain}"
    min_interval = float(redis.get(f"{key}:interval") or 2.0)  # start at 0.5 req/s
    wait = min_interval - (now() - float(redis.get(f"{key}:last") or 0))
    if wait > 0:
        sleep(wait)
    r = requests.get(url, headers=UA, timeout=15)
    if r.status_code == 429:
        redis.set(f"{key}:interval", min_interval * 2)
        retry_after(parse_retry_after(r))
    elif r.ok and min_interval > 0.5:
        redis.set(f"{key}:interval", max(0.5, min_interval * 0.9))
    redis.set(f"{key}:last", now())
    return r
```

Failed jobs go to a **dead-letter queue** with exponential backoff.

### 5.3 Hybrid pagination

Always re-check the newest page of each followed channel/instance for fresh uploads, while separately resuming deep pagination for un-indexed back-catalogs. Track a per-channel `deep_cursor` so interrupted deep crawls resume cleanly.

### 5.4 Tag-backfill pass

List endpoints return sparse tags. A separate job re-fetches `include=63` for tagless videos, tracking progress so the same videos aren't retried forever.

### 5.5 Sync tiers (Celery Beat)

| Tier | Criterion | Interval |
|---|---|---|
| Hot | >10 followers or >3 submissions in 7 days | 2–4 h |
| Active | ≥1 follower or submitted in last 30 days | 6–12 h |
| Cold | no activity in 30 days | 72 h, liveness check first |

### 5.6 Retention

Channel 404 or instance dark → `is_active=false` immediately. Purge videos whose instance hasn't been seen in ~30 days. `video_mirrors` keeps "also available on" links alive through instance outages.

---

## 6. Deduplication

No built-in duplicate detection exists in PeerTube. On insert, match candidate canonical videos by `(title, duration ±5s, published_at ±1h)` across instances; link via `canonical_video_id`, record alternates in `video_mirrors`. Surface "Also available on: [instance list]" on video cards.

---

## 7. Search Service

- Postgres FTS only: `WHERE search_vector @@ plainto_tsquery('english', $1) ORDER BY ts_rank(...)`, plus GiST `similarity(title, $1)` for fuzzy top-N.
- Filters: category, language, duration band, instance, NSFW policy.
- **No Meilisearch/Elasticsearch in v1.** Revisit only with measured evidence (p95 search-as-you-type >200ms, zero-result typo complaints, CJK tokenization demand).
- Phase 2: expose a Sepia-compatible search API so other ecosystem tools can consume our index.

---

## 8. NSFW Handling (PeerTube ≥ v7.2 taxonomy)

NSFW is multi-flag, not binary: `nsfw_flags` JSONB (`violent`, `shocking`, `sexually_explicit`). Site default per-flag policy: **warn** — never silent-hide, per the transparency ethos — overridable per session via cookie (display / warn / blur / hide).

---

## 9. Moderation, Anti-Abuse & Appeals — the actual product

PeerTube 8.3 added **remote mute list** and **word list** subscriptions. Use them from day one.

1. Subscribe to well-known community remote mute lists; auto-import with source attribution in `blocklist.source`.
2. Word lists flag new submissions for human review — humans only see flagged items, never the full firehose.
3. Maintain a local blocklist in the same format so other PeerTube admins can re-subscribe to ours.
4. Every block entry has `appeal_status`; appeals logged anonymized on the public transparency page; **7-day SLA**.
5. **Spam economics:** a channel publishing 20 videos/day floods every chronological feed. Policy: **max 3 videos/channel/day in default and category feeds; overflow lives on the channel page.** Published, uniform, appealable — framed explicitly as *anti-manipulation*, not ranking.
6. Submission hygiene: per-IP-hash rate limit (10/day), CAPTCHA, dedup by canonical identity.
7. Takedown: form + email, action within 48h, soft-delete (`is_active=false`) for audit — never hard-delete.

---

## 10. Thumbnails — proxy and cache, never hotlink

Hotlinking hundreds of concurrent thumbnail requests from our browse pages onto 2-user instances is the opposite of protecting small creators. Proxy thumbnails through our own cache with short TTLs, and honor per-instance `thumbnail_opt_out` instantly.

---

## 11. Frontend (Next.js App Router)

| Page | Strategy | Why |
|---|---|---|
| `/` home, category feeds | ISR 60s | Frequent, not per-request |
| `/search` | SSR / client fetch | Query-dependent, no SEO value in raw results |
| `/channel/{handle}@{domain}` | ISR 300s | Mostly static |
| `/submit`, `/about`, `/transparency` | SSG | Static |
| Video detail | ISR | Fresh metadata + structured data |

**SEO decision:** SSR-for-SEO and cross-domain canonicals can't coexist (Google drops duplicate-canonical pages from its own index). **Ship v1 with Option A (manifesto-pure):** video detail pages exist for UX; `rel=canonical` points to the origin watch URL; growth comes from Fediverse word-of-mouth, not Google. Option B (self-canonical + JSON-LD `VideoObject` + prominent outbound links) is a **config flag**, flipped only by an explicit later decision if organic search becomes a stated goal — don't build half of each.

**UX rules:** every video card links out to the creator's instance; no pulled-back view/like counts anywhere in the schema (the UI literally cannot rank by them); no embedded player by default, respecting uploader embed restrictions; `sitemap.ts`/`robots.ts`; NSFW warn overlays per §8.

---

## 12. Personalization (no accounts)

Browser generates a random `profile_token` in `localStorage`; follows and session preferences attach to it. "My feed" = chronological merge of followed channels. Clearing site data starts a new anonymous profile — accepted trade-off, and arguably on-brand (we hold nothing). Phase 2 (optional): an ActivityPub endpoint so the chronological feed is itself followable from Mastodon.

---

## 13. Deployment, Cost, Operations

| Scale | RAM | ~Cost/mo |
|---|---|---|
| <10K videos | 4GB | ~$20 |
| 10K–100K | 8GB | ~$40 |
| 100K–500K | 16GB | ~$80 |
| 500K+ | 32GB + dedicated search | ~$160+ |

Docker Compose: PostgreSQL, Redis, Celery workers + Beat, Next.js, thumbnail proxy (Caddy/nginx). Backups: nightly `pg_dump` + WAL, 7 daily + 4 weekly, off-site.

**Monitoring:** Uptime Kuma + Sentry, plus per-instance 4xx/5xx rate (sustained 403 → auto-backoff + review flag), `crawl:new` queue depth alert at 100, sync staleness (>2× tier interval), 429 hit rate.

---

## 14. Milestones (honest solo timeline: 10–12 weeks)

The original 6-week MVP silently absorbed a moderation surface (appeals SLA, transparency page, mute-list subscriptions, feed caps) that exceeds that budget. That surface *is* the differentiator — cutting it launches a worse PeerSeek.

- **W1** — Schema (incl. tsvector trigger, NSFW taxonomy), URL parser, submission endpoint. **[Verification battery: rate limits, include bitmask, server/stats endpoint, Retry-After — 5 live instances]**
- **W2** — Celery queues + Beat + Redis token-bucket limiter; single-video crawler; instance registry seeded from joinpeertube.org; robots.txt gate + UA
- **W3** — Channel crawler, hybrid pagination, deep cursors, dead-letter queue
- **W4** — Three-tier sync; liveness + retention jobs; tag backfill
- **W5** — Search API (GIN/GiST); Next.js browse + channel + video pages (ISR)
- **W6** — NSFW warn overlays; remote mute-list subscription; word-list flagging; review queue
- **W7** — Thumbnail proxy + opt-out; per-channel feed caps
- **W8** — Profile tokens + follows + "my feed"
- **W9** — Transparency page, appeals queue, bot-policy page, positioning/comparison page
- **W10** — Seed ~20 known educational instances; hardening; launch (W11–12 buffer)
- **Phase 2** — Sepia-compatible search API; optional ActivityPub endpoint; multilingual UI
- **Phase 3** — Meilisearch only on measured latency wall; thumbnail CDN

---

## 15. Risk Register

| Risk | Mitigation |
|---|---|
| Differentiation failure vs Sepia/PeerSeek/Fedi.Video | Lead with transparent appealable rules + chronological-only ordering; publish comparison page before writing crawler code |
| Curation burden kills the project (PeerTube Index precedent) | Remote mute lists + word lists from day one; humans review flagged items only |
| Chronological feed spam-flooding | Public per-channel feed caps (§9.5), framed as anti-manipulation policy |
| Instance hostility (429s, robots blocks) | Descriptive UA, robots compliance, adaptive token bucket, public bot policy |
| Stale index | `is_active` flags + 30-day dark-instance purge + mirrors |
| Thumbnail load on small instances | Proxy/cache with instant opt-out (§10) |
| SEO self-contradiction | Option A shipped deliberately; Option B behind an explicit flag (§11) |
| Legal (thumbnails, EU) | Metadata-only + proxied thumbs; 48h takedown; soft-delete audit trail |
| Scope creep | Cut Meilisearch, accounts, Kubernetes for v1 — tokens replace accounts, Compose scales to ~500K videos |
| Timeline underestimation | 10–12 week honest budget; moderation surface is non-negotiable scope |

---

## 16. Definition of Done (v1)

- [ ] Submit a video URL or a channel; metadata appears without human approval
- [ ] Search returns ranked results <200ms p95 at 50K videos
- [ ] Default feeds are chronological with published per-channel caps
- [ ] Every moderation action appears on the public transparency page with reason + appeal status
- [ ] Appeals answered within 7 days
- [ ] robots.txt and opt-out honored; descriptive UA published
- [ ] No engagement metric influences any ordering, anywhere
- [ ] All code open; index exportable; Sepia-compatible API spec'd (implementation phase 2)

## 17. Immediate Next Steps

1. Probe 5 live instances (Week 1): real rate limits, `include` bitmask, `Retry-After` behavior, `server/stats` endpoint.
2. Talk to 2–3 instance admins and the Fedi.Video curator before finalizing crawl/embed policy.
3. Write and publish the comparison/positioning page — spam policy stated up front.
4. Start building. No further plan-review rounds — the open questions left are empirical, not architectural.
