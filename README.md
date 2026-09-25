# Open Video Discovery for PeerTube

A transparent, non-hosting discovery index for PeerTube: search and browse metadata for videos that stay on their creators' own instances, ordered chronologically — never by engagement.

**Status:** Pre-code — architecture and technical plan finalized, build not yet started. See [Milestones](TECHNICAL_PLAN.md#14-milestones-honest-solo-timeline-10–12-weeks).

[![License](https://img.shields.io/github/license/titouhalloub/Open-Video-Discovery-For-PeerTube)](LICENSE)
[![Status](https://img.shields.io/badge/status-planning-yellow)](TECHNICAL_PLAN.md)

---

## What this is

We collect **metadata only** — title, description, thumbnail, watch link — for videos hosted on PeerTube instances, and make it searchable in one place. We never host, re-upload, or modify a video. Every result links back to the creator's own server.

- **No engagement ranking.** Feeds are chronological or manually curated — never sorted by views, likes, or watch time.
- **No accounts.** Follows and preferences are tied to an anonymous local token, not a login.
- **Transparent moderation.** Every removal or block is logged publicly with a reason, and is appealable within a published SLA.
- **Creator control stays put.** Submit a single video or a whole channel; nothing is copied, only indexed.

Full rationale for why this exists is in the [Manifesto](#manifesto--project-overview) below. Full architecture, data model, API integration, and build plan are in **[TECHNICAL_PLAN.md](TECHNICAL_PLAN.md)**.

## Tech stack

Python (FastAPI + Celery) · PostgreSQL (full-text search, no external search engine) · Redis · Next.js · Docker Compose. Full rationale for each choice is in [§1 of the technical plan](TECHNICAL_PLAN.md#1-core-decisions).

## Contributing

This project is pre-code and welcomes design feedback as much as code:

- **Developers:** the technical plan's [§17 Immediate Next Steps](TECHNICAL_PLAN.md#17-immediate-next-steps) is the current entry point — instance API verification and crawler groundwork come first.
- **PeerTube instance admins:** feedback on crawl etiquette and embed policy is explicitly wanted before the crawler is built — open an issue.
- **Everyone else:** read the manifesto below and open an issue if something about the moderation policy or scope doesn't sit right. Every decision here is meant to be discussable.

See also [Advertising & Sponsorship.md](Advertising%20%26%20Sponsorship.md) for how the project intends to fund itself without selling attention.

## License

See [LICENSE](LICENSE).

---

## Manifesto & Project Overview

### Why this project exists

This project was created to explore alternative, open, and transparent ways to index and discover video content in a federated ecosystem. We didn't start it because we have funding, a large team, or a marketing plan.

We started it because we saw something quietly breaking.

For years, many people tried to share simple knowledge — someone explaining how to fix a device, teaching a craft, or sharing an experience that could help others. These channels were calm, educational, and not designed to make money.

Yet algorithms didn't notice them. Not because the content was bad, but because it wasn't profitable. Creators learned a harsh lesson: if you don't produce what the algorithm loves, you don't exist. Some withdrew. Some adapted. Many remained unheard.

Meanwhile, platforms filled with recycled videos, clickbait, soulless AI-generated content, and material targeting children with no regard for ethics or education. The question stopped being "is this content useful?" and became "will it keep the user engaged the longest?" That's the break that inspired this project.

### What this project does

We are building a single centralized discovery website that:

- Does **not** host videos
- Does **not** re-upload videos
- Does **not** change or control videos

It simply collects metadata — title, description, thumbnail, watch link — organizes it, and makes it discoverable. Videos remain on the creator's server, under the creator's full control, on their own terms.

Creators can submit a single video link or their entire channel. The system uses PeerTube's public APIs to fetch and update information automatically — no manual approval, no human interference, no hidden algorithm.

### What users see

A simple interface: search for videos or channels, browse categorized content, follow channels, and click through to watch **directly from the creator's server**. We are a pass-through point, not a destination.

### What we do NOT do

- We do not own videos
- We do not place ads
- We do not impose hidden algorithms
- We do not bury content because it's not profitable
- We do not ask creators to change their style

Content can be ordered chronologically, manually, or by user preference — never based on engagement metrics alone.

### Who this project is for

Small educational creators; calm and constructive content; videos often ignored by commercial platforms; audiences looking to discover, not just consume.

### Why centralized?

Users want one place to find content, not dozens of scattered websites — centralized *discovery*, not centralized *control*. If this site fails one day, the code is open and the idea can be replicated. Nothing is lost.

### How the project is managed

Fully open-source, transparent funding (donations/community support), no data selling, every decision discussable.

### Future potential

If successful: better filters and search, multiple interface options, language and regional support. If it fails: it remains a genuine attempt to cut through the noise.

### Final word

This project doesn't ask anyone to leave YouTube. It doesn't claim to be the alternative. It simply says: there is content worth seeing, and we try to shine a light on it.

Anyone who sees themselves here is welcome — as a user, a creator, a developer, or a critic.
