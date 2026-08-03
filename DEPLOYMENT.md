# Deployment architecture (decided)

*Proposed and decided 2026-08-03. Status: **decided, build not started.***

Goal: Entrio live on **entrio.ca**, scalable to 1000+ hosts, run **cheaply and
predictably**. The cloud shape is settled; the local upgrades follow from it.

## Decisions (2026-08-03)

| Decision | Choice | Why |
|---|---|---|
| Deployment shape | **Always-on container** | Predictable flat cost, low ops, the app is a long-lived `next start` server |
| Host | **Render** | Managed Postgres + cron + disks + TLS in one dashboard, git-push deploys, least ops |
| Sequencing | **Postgres first, then deploy** | First deploy is multi-instance-ready; no second migration; also drops Litestream entirely |
| Photo storage | **Cloudflare R2** | Zero egress fees on image serving, S3-compatible (portable), 10 GB free |
| Postgres provider | **Decide at connection time** | Migrate provider-agnostic (standard SQL); pick Neon vs Render Postgres when we wire the connection string |

Rejected: **Vercel/serverless** (usage-based cost, ~$20/mo/seat floor, forces
the same migration up front) and **Phase-0-with-SQLite** (would mean a second
migration later — the user chose to do it once, properly).

## The decided architecture

```
              entrio.ca (GoDaddy DNS → CNAME → Render)
                        │  TLS auto
        ┌───────────────▼───────────────┐
        │   Render web service           │
        │   next start (standalone)      │   1 instance now
        │   1 → N instances = scale-out  │   +$7/mo per extra
        └───┬───────────┬────────────┬───┘
            │           │            │
    ┌───────▼──┐  ┌─────▼─────┐  ┌───▼──────────┐
    │ Postgres │  │ R2        │  │ Render Cron  │
    │ Neon or  │  │ (photos)  │  │ → /api/      │
    │ Render PG│  │ S3 SDK    │  │   calendars/ │
    │ managed  │  │ + CDN     │  │   sync       │
    │ backups  │  └───────────┘  └──────────────┘
    └──────────┘
    Pass-through APIs: Stripe · Google Places · Anthropic · Resend
```

Multi-instance-safe from day one: state in Postgres (not process memory),
photos in R2 (not local disk), backups managed by the DB provider. No
Litestream, no persistent disk to be a single point of failure.

## Build order (local work, in sequence)

### 1. Postgres migration — the big piece *(gating everything)*
The SCALING.md Stage 2 work, moved to the front. Rewrite the ~30 `store.ts`
functions as **scoped SQL queries** behind the unchanged contract; normalise
into indexed tables (`accounts`, `properties`, `reservations`, `guests`,
`upsells`, `tasks`) keyed and indexed on `(account_id)`, `(portal_token)`,
`(external_id)`; retire the in-memory `Db`, `build()`, `invalidate()` and the
overlay-replay model **for real data** (keep seed-replay for the demo only).
- Written **provider-agnostic** (standard SQL, a connection pool, no
  Neon-specific features) so Neon or Render Postgres both work.
- Dev: run Postgres locally (Docker) or point at a free Neon branch.
- The store *contract* is unchanged, so app code and the 14 test suites are the
  guard that the migration preserved behaviour. **`tenant-isolation`,
  `door-gate`, and `audit:guest` must stay green throughout** — they're the
  proof the swap didn't leak or cross tenants.
- This also dissolves most SCALING.md soft walls (indexed token lookup,
  SQL-side upsell counts) and lets throttle/sync locks move into the DB.

### 2. Photos → R2 *(one file + config)*
`api/photos/route.ts` writes to R2 via the S3 SDK; the returned `src` becomes
the R2/CDN URL. Existing `/photos/*` seed assets can stay bundled or move to R2
too.

### 3. Containerize for Render
`output: "standalone"` in `next.config`, a `Dockerfile` (or Render's native
Node build), healthcheck, and env-var wiring. No app-logic change.

### 4. Deploy + connect
Render web service → Postgres (provider picked here) → R2 bucket → Render Cron
hitting `/api/calendars/sync` with `ENTRIO_CRON_SECRET` → GoDaddy DNS to Render
→ auto TLS. Real Stripe webhook endpoint + signing secret (not the CLI's).

### 5. Later (Phase 2, when a single instance strains)
Scale to 2+ instances; staggered queued calendar jobs; Sentry + structured
logs; webhook `event.id` dedupe.

## What the user needs to set up (accounts / credentials)

- **Render** account (app + cron).
- **Cloudflare** account → an R2 bucket + API token.
- **Postgres**: a Neon project *or* a Render Postgres instance (decided at
  step 4). A connection string either way.
- **GoDaddy**: point entrio.ca (currently a parked A record) at Render.
- **Env vars in Render**: everything in INTEGRATIONS.md, plus `DATABASE_URL`,
  R2 credentials, and `ENTRIO_SITE_URL=https://entrio.ca`.
- **Stripe**: a live webhook endpoint + its signing secret.

## Cost (decided shape)

| Item | Start | ~hundreds of hosts |
|---|---|---|
| Render web service | ~$7 | $7–21 (1–3 instances) |
| Postgres (Neon free → paid) | $0 | ~$19 |
| R2 (photos) | ~$0–1 | ~$1–5 |
| Email (Resend free 3k/mo) | $0 | $0 |
| **Fixed monthly** | **~$7–8** | **~$30–45** |

Usage-based and small, scaling with use not host count: Stripe (% per txn),
Google Places (~$0.028 per "Recommend" click — verify current free-credit
terms), Anthropic Haiku (pennies/draft). Under ~$100/mo well past 100 hosts.
