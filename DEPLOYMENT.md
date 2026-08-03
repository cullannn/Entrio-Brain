# Deployment architecture (proposal)

*Drafted 2026-08-03. Status: **awaiting decisions** (see end). Not yet built.*

Goal: Entrio live on **entrio.ca**, scalable to 1000+ hosts, run **cheaply and
predictably** at the current tiny scale. This proposal picks the cloud shape,
then the local upgrades follow from it.

## The one decision everything hangs on

Entrio is a long-lived `next start` server that today writes to exactly two
places on disk: the SQLite overlay (`.data/`) and photo uploads
(`public/photos/uploads`). Everything else reads. That makes the fork clean:

**A. Always-on container** (Render / Fly / Railway) — one persistent Node
process with a disk. The app runs *almost as-is*: SQLite on the disk, photos
on the disk. Flat monthly cost. Scales by adding instances **once** state moves
to Postgres. **Predictable, cheap, live this week.**

**B. Serverless** (Vercel) — best-in-class Next.js DX, effortless autoscale,
scale-to-zero. But: no persistent filesystem, so **both** disk writes must move
to Postgres + object storage *before you can deploy at all*; commercial use
starts at ~$20/mo/seat; and cost is usage-based (less predictable as traffic
grows).

**Recommendation: A, always-on container.** It matches "cost-cautious +
predictable + get live soon", lets us deploy before the Postgres migration
(de-risking the launch), and still scales horizontally after that migration —
which is the same migration Vercel would force up front. We don't give up
scalability by choosing A; we just sequence it sanely.

## Recommended architecture

```
              entrio.ca (GoDaddy DNS → CNAME)
                        │  TLS auto
        ┌───────────────▼───────────────┐
        │   App host (always-on)         │   Render web service (rec.)
        │   next start, 1 instance now   │   ~512MB–1GB, flat $/mo
        │   → 2+ instances after Postgres│   scale-out = add instances
        └───┬───────────┬────────────┬───┘
            │           │            │
    ┌───────▼──┐  ┌─────▼─────┐  ┌───▼──────────┐
    │ Postgres │  │ Object    │  │ Cron trigger │  platform scheduler
    │ (Phase 1)│  │ storage   │  │ → /api/      │  → calendar sync
    │  Neon    │  │ R2 (photos)│ │   calendars/ │
    └──────────┘  └───────────┘  │   sync       │
                                 └──────────────┘
    Pass-through APIs (usage-based, scale with use not hosts):
      Stripe · Google Places · Anthropic (Haiku) · Resend (email)
```

## The phased path — build only what scale demands

### Phase 0 — Live on entrio.ca, single instance (days)
Deploy the app as-is to an always-on host with a **persistent disk**. SQLite
stays; photos move to **R2** (cheap, durable, irreplaceable data off the
single disk — the upload route already returns a `src`, so it's a small
change). **Litestream** streams the SQLite file to R2 continuously, so a lost
disk is minutes of data, not everything.
- One instance is genuinely fine for the first tens of hosts (low-traffic,
  correctness-critical app, not high-QPS).
- Gets the product real and backed-up before touching the store internals.
- **~$8–10/mo.**

### Phase 1 — Postgres behind the store (1–2 weeks), when scale demands it
The SCALING.md Stage 2 work: normalise into indexed tables, make each store
function a scoped query, retire the in-memory world model for real data (keep
seed-replay for the demo). The store *contract* doesn't change, so app code
doesn't. This is what unlocks **multiple instances and zero-downtime deploys**;
until then the single instance is the ceiling.
- Provider recommendation: **Neon** (serverless Postgres, scale-to-zero, real
  free tier, preview-branch per deploy). Alternative: the host's built-in
  Postgres (one dashboard, simpler) or Supabase (Postgres + extras you don't
  need yet).
- **+$0 on Neon free tier → ~$19/mo when outgrown.**

### Phase 2 — Scale-out (only when a single instance strains)
Run 2+ app instances behind the platform's load balancer; move the per-process
throttle/sync locks into Postgres (SECURITY.md known gaps); calendar sync
becomes staggered queued jobs; add Sentry + structured logs.
- **+~$7/mo per extra instance.**

## Cost picture (honest, at your stage)

| Item | Phase 0 | At ~hundreds of hosts |
|---|---|---|
| App host (Render Starter) | ~$7 | $7–21 (1–3 instances) |
| Persistent disk / — | ~$0.25 | (retired at Phase 1) |
| Postgres (Neon) | — | $0 free → ~$19 |
| Object storage (R2) | ~$0–1 | ~$1–5 |
| Backups (Litestream→R2) | ~$0 | (Postgres-managed) |
| Email (Resend) | $0 free | $0 free (3k/mo) |
| **Fixed monthly** | **~$8–10** | **~$30–50** |

Usage-based, scale with *use* not host count, all small: **Stripe** (% of
transactions, no fixed fee) · **Google Places** (~$0.028 per host "Recommend"
click — verify current free-credit terms, Google changed them in 2025) ·
**Anthropic** Haiku (pennies/draft) · these are effectively pass-through.

You stay under ~$100/mo well past your first hundred hosts.

## What each choice changes in the local code

- **Host = always-on container**: add `output: "standalone"` to `next.config`
  (smaller image), a `Dockerfile` or the host's buildpack, env-var wiring, and
  a healthcheck. No app-logic change.
- **Photos → R2**: `api/photos/route.ts` writes to R2 via the S3 SDK instead of
  local disk; the returned `src` becomes the R2/CDN URL. One file.
- **Litestream**: a sidecar config; no code change.
- **Postgres (Phase 1)**: rewrite the ~30 store functions as queries behind the
  unchanged contract; retire `build()`/`invalidate()`/in-memory `Db` for real
  data. The biggest piece, isolated to `store.ts` + `overlay-db.ts`.

## Decisions needed (see the questions posed alongside this doc)

1. **Deployment shape & host** — always-on container (rec.) vs serverless; and
   which host.
2. **Go-live sequencing** — Phase 0 now with SQLite+disk (rec.), or do Postgres
   first before any deploy.
3. **Object storage** — Cloudflare R2 (rec.) vs S3 vs host-native.
4. **Postgres provider** (for Phase 1) — Neon (rec.) vs host built-in vs
   Supabase. Can be deferred.

Once chosen, this doc is rewritten from *proposal* to the *decided*
architecture, and the local upgrades begin in phase order.
