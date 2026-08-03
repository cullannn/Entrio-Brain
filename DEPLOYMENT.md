# Deployment architecture (decided)

*Decided 2026-08-03. Status: **steps 1–2 (Postgres, R2 photos) done; steps 3–4 (containerize, deploy) pending.***

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

### 1. Postgres migration — the big piece *(DONE 2026-08-03)*
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

### 2. Photos → R2 *(DONE 2026-08-03)*
A `storage()` provider (same shape as `payments()`/`billing()`): R2 when all
five `R2_*` vars are set, the local `public/` folder otherwise. `api/photos`
calls `storage().put()` and stores the src it returns; uploads are keyed per
account (`uploads/<accountId>/<file>`). Seed assets stay bundled — only host
uploads need durable storage, since the container wipes `public/` on redeploy.
- **Not the S3 SDK.** `aws4fetch` (~2KB) signs; the PUT goes over **`node:https`**,
  not `fetch` — R2 needs Content-Length and Next.js patches global fetch to send
  byte bodies chunked with none (411 MissingContentLength). See INCIDENTS.
- Verified against a live bucket: upload → 200 with an absolute `pub-….r2.dev`
  URL → object publicly served, byte-identical, `immutable` cache header.
- Serving needs no config: `Photo` is a plain `<img>`, so absolute URLs and the
  existing `/photos/*` paths both just work.

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

## AWS later? Portability, reliability, and the exit path

*Discussed 2026-08-03. Not a decision to move — a note on why the door stays open.*

**Is AWS more reliable at scale?** Higher *ceiling*, not automatically a higher
*floor*. Render, Neon and R2 are all production-grade (R2 11-nines durability;
Neon separated storage/compute with automated failover + backups; Render runs
on AWS/GCP underneath) and comfortably deliver ~99.9%+ at hundreds-to-low-
thousands of hosts. AWS's edge is *extreme* HA — active-active multi-region,
sub-minute failover, five-nines, strict data-residency/compliance — and only
when you invest the ops to build and test it; a badly-run AWS deploy is *less*
reliable than a well-run Render/Neon one. The stack also already spreads risk
across three vendors (app/Render, data/Neon, photos/Cloudflare), so one
provider's outage doesn't take everything. **Move to AWS for a concrete need
(a contractual SLA, multi-region, compliance, or sustained large-scale traffic
where reserved-instance discounts flip the cost math) — not merely "more
hosts," which is a scaling knob Render/Neon handle well past the first
thousand.**

**Is the migration easy?** Yes, by design — it's config, not a rewrite:
- **App**: a standard `next start` container → App Runner or ECS/Fargate. Same
  image, near lift-and-shift.
- **Postgres**: `pg_dump` Neon → `pg_restore` RDS/Aurora, change `DATABASE_URL`.
  **Zero app-code changes** — the store is provider-agnostic standard SQL, no
  Neon-specific features on purpose.
- **Photos**: R2 is S3-compatible (already the S3 SDK). Sync the bucket, change
  endpoint + credentials. Config, not code.
- **Cron/email**: point EventBridge at `/api/calendars/sync`; Resend works
  anywhere (or swap to SES).

The real work is the AWS *scaffolding* (VPC, subnets, security groups, IAM,
secrets, CloudFront, monitoring) — a few days of one-time setup, plus a DNS
cutover. **This is exactly why container + provider-agnostic-SQL + S3-compatible
storage were the right first move: live cheaply and simply now, AWS door wide
open, no lock-in.** (Vercel-serverless would have made this move much harder —
another reason it was rejected.)

**To keep portability:** stay on standard, portable primitives — no Neon
branching/HTTP-driver in app code, no hard dependence on Render-only features,
nothing R2-specific beyond the S3 API.

**Rough cost contrast** (US regions, low traffic, estimates): decided stack
~$7–8/mo start → ~$30–45 at hundreds of hosts. AWS managed equivalent (App
Runner + RDS + S3/CloudFront) ~$40–75/mo start — driven by RDS having no
perpetual free tier (Neon does), S3 egress fees (R2 has none), and the NAT-
gateway trap (~$32/mo for private RDS access). AWS bare-EC2 is ~$13–20/mo but
trades the dollars for ops you'd do yourself. AWS wins on price only at large
scale with committed-use discounts.
