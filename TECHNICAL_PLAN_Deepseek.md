```markdown
# Technical Plan: Open Video Discovery for PeerTube

## 1. Strategic Positioning & Differentiation

This is not "another PeerTube search engine." Sepia Search (search), PeerSeek (browse/discovery), and Fedi.Video (human curation) already cover large parts of that space.

**What none of them do well:** a transparent, appealable, rule-based moderation process combined with chronological-only ordering and a public anti-manipulation policy.

**Differentiators:**

- Public moderation log + appeals queue with SLA.
- No engagement metrics, no algorithmic ranking.
- Per-channel feed inclusion rate limits, published and applied uniformly.
- Creator-submitted + crawl-supported, with a clear opt-out path.
- Sepia-compatible search API surface (ecosystem interoperability).

**Action before code:** Write the public comparison/positioning page. It must explain the spam policy upfront.

## 2. Core Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Video hosting | None — pass-through | Creators keep control. |
| Data stored | Metadata only | Minimal legal/ethical surface. |
| Ordering | Chronological + manual curation + token-based follows | No engagement ranking. |
| Anti-spam | Public per-channel feed inclusion rate limits | Chronological feeds are spam-exploitable; rate limits are anti-manipulation policy, not ranking. |
| Federation | Pull-based PeerTube REST API | Simpler than ActivityPub. |
| Search | PostgreSQL FTS only | PeerSeek proves Postgres FTS handles federation scale (~940K videos). No Meilisearch until measured need. |
| User accounts | Anonymous profile tokens (localStorage) | Follows/prefs without accounts, GDPR, or password security. |
| Thumbnails | Proxy/cache with short TTL, per-instance opt-out | Kinder to small instances than hotlinking. |
| SEO | Self-canonical on discovery pages; prominent outbound links | Deliberate choice: rank for discovery intent, not watch intent. Resolves SSR vs. canonical contradiction. |

## 3. Tech Stack

- **Backend:** Python FastAPI + Celery
- **Database:** PostgreSQL + `pg_trgm` + `tsvector`
- **Cache/queue:** Redis
- **Frontend:** Next.js App Router + Tailwind
- **Reverse proxy:** Caddy or nginx
- **Deploy:** Docker Compose
- **Explicitly not in MVP:** Kubernetes, Meilisearch, user accounts with passwords.

## 4. Architecture

```
Creator submits URL ──► Submission API ──► Validator ──► Celery queues
                                                        │
                                                        ▼
                                              PeerTube Instance API
                                              (/api/v1/videos, /api/v1/video-channels/{name})
                                                        │
                                                        ▼
                                              PostgreSQL (metadata DB)
                                                        │
                          ┌─────────────────────────────┼─────────────────────────────┐
                          ▼                             ▼                             ▼
                   Public Search API           Token-based Follow/Feed       Admin/Curation Panel
                          │
                          ▼
                     Next.js Frontend
                   (search, browse, follow, watch-links out)
```

**Key components:**

1. Submission service — validates URLs, resolves instance domain, normalizes, deduplicates.
2. Crawler worker pool — Celery queues by priority: `crawl:new`, `crawl:sync`, `crawl:liveness`, `crawl:deep`.
3. Search service — PostgreSQL FTS with `search_vector tsvector` + GIN, `pg_trgm` GiST for fuzzy title.
4. Follow/feed service — anonymous profile tokens, no accounts.
5. Admin panel — remote mute list subscriptions, word lists, appeals queue, public transparency log.

## 5. Data Model

```
instances        (id, domain, name, version, last_crawl_at, is_blacklisted)
channels         (id, instance_id, handle, display_name, description, avatar_url,
                  followers_count?, submitted_at, last_synced_at, is_active)
videos           (id, channel_id, uuid, peer_tube_video_id, title, description,
                  thumbnail_url, embed_url, watch_url, duration, tags[],
                  category_id, language, published_at, last_synced_at, is_available,
                  nsfw_flags JSONB, nsfw_policy TEXT, search_vector tsvector,
                  canonical_video_id)
video_mirrors    (video_id, instance_id, watch_url, last_seen_at)
categories       (id, name, slug)
submissions      (id, submitted_url, type, status, submitter_hash, created_at)
user_follows     (profile_token, channel_id, created_at)
appeals          (id, video_id, instance_id, reason, status, created_at, resolved_at)
blocklist_subs   (id, source_url, last_synced_at)
word_lists       (id, source_url, last_synced_at)
crawl_jobs       (id, instance_id, channel_id, status, priority, created_at)
dead_letter      (id, job_id, error, created_at)
```

**Important details:**

- Store PeerTube `uuid` + instance domain; reconstruct watch URL as `https://{domain}/w/{shortUUID}` or `/videos/watch/{uuid}`.
- NSFW is multi-flag: `{"violent": bool, "shocking": bool, "sexually_explicit": bool}`. Default site policy: `warn`.
- Deduplication: `canonical_video_id` + `video_mirrors`. Match on `(title, duration ±5s, published_at ±1h)` across instances.
- Search vector populated by trigger on insert/update; GIN index. `pg_trgm` GiST on title for fuzzy.

## 6. PeerTube API Integration

### Endpoints

- Video: `GET /api/v1/videos/{id}` → title, description, thumbnail path, tags, category, duration, publishedAt, account, NSFW flags.
- Channel: `GET /api/v1/video-channels/{name}` → metadata + `videosCount`.
- Channel videos: `GET /api/v1/video-channels/{name}/videos` (paginated, `count`/`start`).
- Server stats: `GET /api/v1/server/stats` for cheap instance liveness.

### URL handling

Normalize all variants to `(domain, uuid)`:
- `/w/{shortUUID}` (22-char)
- `/videos/watch/{uuid}` (36-char)
- `/videos/embed/{uuid}`

### Rate limiting

- Default is ~50 req/10s, but instance-configurable. Do not hardcode. The default `rates_limit.api` is `max: 50` per 10-second window.
- Adaptive per-domain token bucket, start at 0.5 req/sec, adjust up/down on 429/5xx.
- Honor `Retry-After` (seconds and HTTP-date).
- Redis tracks `ratelimit:{domain}`.

### Pagination & crawling

- `start`/`count` offset pagination, max 100.
- `include=1` for cheap discovery; `include=63` for full sync. `include=63` means "include all possible fields". Verify bitmask on live instances in Week 1.
- **Hybrid strategy:** always re-check newest page for fresh uploads; resume deeper pagination for incomplete back-catalog.
- **Tag-backfill job:** separate pass for sparse tags; track progress to avoid retrying tagless videos.
- **Sync tiers:**
  - Hot (>10 followers or >3 submissions/7d): every 2–4h.
  - Active (≥1 follower or submitted in 30d): every 6–12h.
  - Cold (no activity 30d): every 72h, liveness check first.
- **Dead-instance retention:** `is_active` flag + purge after N days dark.
- **Dead-letter queue:** failed jobs with exponential backoff.

### robots.txt & user-agent

- Descriptive UA: `OpenVideoDiscoveryBot/1.0 (+https://yourdomain.org/bot)`.
- Published bot policy page.
- Respect `robots.txt` per instance.
- Opt-out mechanism via submission-blocklist endpoint.

## 7. Anti-Abuse & Moderation

- **Remote mute list subscriptions** (PeerTube 8.3 official term) + word lists. Subscribe from day one. PeerTube 8.3 allows admins to subscribe to remote block lists that are periodically updated and notify admins when an account or platform is blocked.
- **NSFW:** multi-flag taxonomy, default `warn`, per-session override via cookie.
- **Deduplication:** canonical video + mirrors.
- **Rate limiting:** per-IP submission limits (e.g., 10/day), CAPTCHA on submission.
- **Copyright/takedown:** form + email; removal sets `is_active=false`; public log.
- **Appeals queue:** public anonymized log; 7-day SLA.
- **Transparency page:** blocked instances/videos, reasons, appeals.
- **Public anti-manipulation policy:** per-channel feed inclusion rate limits (e.g., max N videos/channel/day in default feeds). Overflow on channel page. Policy is public, same for everyone. This preempts "rate limits are an algorithm" criticism.

## 8. Frontend Plan (Next.js)

### Pages

- `/` — chronological browse + curated categories (default newest).
- `/search?q=` — FTS with filters (category, language, duration, instance).
- `/channel/{handle}@{domain}` — channel page.
- `/submit` — URL submission.
- `/follows` — token-based followed channels feed.
- `/about`, `/transparency`, `/instances`, `/bot`, `/comparison`.

### UX rules

- Every video card links out to creator's instance. No embedded player by default.
- No like counts, view counts, engagement metrics.
- Categories from PeerTube's 15 fixed categories + admin-curated collections.

### Rendering strategy

| Page | Strategy |
|---|---|
| `/` | ISR 60s |
| `/search` | SSR or client-side fetch |
| `/channel/...` | ISR 300s |
| `/submit` | SSG |
| `/about`, `/transparency` | SSG |
| Video detail | SSR or ISR |

### SEO

- **Decision:** self-canonical on discovery pages. We rank for discovery intent, not watch intent. Prominent outbound links to origin.
- JSON-LD `VideoObject` on video detail pages.
- `sitemap.ts`, `robots.ts`.
- No cross-domain canonical to origin.

### Anonymous profile token

- Browser generates random `profile_token`, stored in `localStorage`.
- Follows and NSFW preferences associated with token.
- Clear by clearing site data.

## 9. Build Milestones (12 weeks solo)

**Weeks 1–2:** DB schema, URL parser, submission endpoint, probe 5 live instances for real API behavior. Celery setup, queues, per-domain rate limiting, single-video crawler, instance registry seeded from `instances.joinpeertube.org` (API endpoint: `https://instances.joinpeertube.org/api/v1/instances?count=1000&start=0`).

**Weeks 3–4:** Channel crawler + pagination, three-tier sync, liveness checks, tag-backfill. Search API (Postgres FTS + GIN + GiST), Next.js browse + video detail pages.

**Weeks 5–6:** NSFW filtering, remote mute list + word list subscription, rate limiting, Docker Compose. Transparency page, appeals queue, bot policy, comparison page.

**Weeks 7–10:** Seed with 20 educational instances. Deduplication + mirrors. Retention job. Polish. Monitoring.

**Weeks 11–12:** Testing, load testing, launch.

**Phase 2:** ActivityPub endpoint (optional, followable feed), multilingual UI.

**Phase 3:** Meilisearch only if measured latency wall. Thumbnail CDN.

## 10. Deployment & Cost

| Scale | RAM | Est. monthly |
|---|---|---|
| <10K videos | 4GB | ~$20 |
| 10K–100K | 8GB | ~$40 |
| 100K–500K | 16GB | ~$80 |
| 500K+ | 32GB + dedicated search | ~$160+ |

- Backups: nightly `pg_dump` + WAL archiving; 7 daily + 4 weekly.
- Monitoring: Uptime Kuma, Sentry, crawl success rate per instance, queue depth, sync staleness, 429 rate.

## 11. Risk Register

| Risk | Mitigation |
|---|---|
| Differentiation failure | Lead with transparent, appealable moderation + chronological-only + public anti-manipulation policy. Write comparison page first. |
| Curation burden kills project | Subscribe to remote mute lists + word lists from day one. Automate flagging. Human review only for flagged items. |
| Instance hostility | Descriptive UA, robots.txt compliance, adaptive token bucket, public bot policy. |
| Stale index | Retention job + liveness tiers. |
| Legal (thumbnails, EU) | Proxy/cache with short TTL, per-instance opt-out. Honor takedowns <48h. |
| Scope creep | Cut Meilisearch, accounts, Kubernetes. Postgres FTS is enough; tokens replace accounts; Compose until ~500K videos. |
| Chronological spam flood | Public per-channel feed inclusion rate limits. |
| SEO contradiction | Self-canonical on discovery pages; link out prominently. |

## 12. Key Risk to Flag Early

The "no algorithm" positioning needs one explicit exception: spam/illegal content filtering and anti-manipulation rate limits. Discovery still needs some ordering. This plan keeps ordering chronological/manual and makes filtering transparent and appealable. State this on the site before launch.

## 13. Immediate Next Steps (Empirical)

1. Probe 5 live instances in Week 1: real rate limits, `include` bitmask, `Retry-After` usage.
2. Talk to 2–3 instance admins and the Fedi.Video curator before finalizing crawl policy.
3. Write the public comparison/positioning page. Put the spam policy upfront.
4. Consolidate `TECHNICAL_PLAN.md` (this document).
5. Start building. No fourth review round.
```
