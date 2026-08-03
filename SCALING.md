# Scaling to 1000+ hosts

*Assessment date: 2026-08-03. Re-assess after Stage 2 lands.*

## Verdict

The **contract** scales; the **implementation** doesn't. Every store function
takes `accountId` first and the app never reaches around that interface, so
the path to 1000 hosts is swapping what's behind `store.ts` — not rewriting
the app. The discipline held in an adversarial review; it's the codebase's
most valuable property.

## Hard walls (break, not degrade)

1. **Single process, whole world in RAM.** `db()` caches the entire built
   dataset on `globalThis`. ~200k reservations at target scale → hundreds of
   MB, rebuilt on every `invalidate()`. Worse: **two instances is broken,
   not risky** — instance A's cache never learns of B's writes; permanent
   stale reads. No horizontal scale, no zero-downtime deploys, one crash
   takes everyone.
2. **The calendar cron can't finish.** `/api/calendars/sync` iterates every
   account × property × feed serially under `maxDuration = 60`. ~2000 feeds
   × ~1 s ≫ 60 s: the tail of the account list simply never syncs, silently,
   every hour.
3. **Photo uploads on local disk** (`public/photos/uploads`): lost on
   redeploy on managed platforms, invisible to a second instance, and no
   durability story — it's the only copy of hosts' photographs.
4. **Per-process security state**: sign-in throttle and sync locks are
   module-level Maps. N instances = N× allowed password attempts and
   duplicate syncs.

## Soft walls (degrade)

Full-array scans per read (`getReservations`, `getReservationByToken`) ·
`availableUpsells` O(upsells × reservations) per portal view ·
`accountForToken` hashes every account per unauthenticated reset call ·
no Stripe event-id dedupe (benign today; handlers are idempotent) ·
observability is `console.error` · **no backups of `.data/`**.

## Already scales — leave alone

Stateless HMAC sessions · tenancy discipline · demo data off in production ·
row-diffed SQLite writes · force-dynamic SSR over cheap in-memory reads.

## The plan

**Stage 1 — before real multi-host traffic (days):**
1. **Backups now.** Litestream (or cron `sqlite3 .backup`) replicating
   `.data/overlay.db` off-box. Highest value hour available.
2. **Batch the cron**: oldest-`lastImportedAt` first with a ~45 s time
   budget; the hourly cadence works through the backlog. "Never syncs the
   tail" → "every feed within a few hours", no new infra.
3. Webhook `event.id` dedupe table; reset-token digest lookup.

**Stage 2 — the real one (1–2 weeks): queried database behind the store.**
Normalise into tables (accounts, properties, reservations, guests, upsells,
tasks) indexed on `(account_id)`, `(portal_token)`, `(external_id)`; each
store function becomes a scoped query; retire the in-memory `Db`, `build()`,
`invalidate()` and patch-replay for real data (keep seed replay for the
demo). **Managed Postgres** (Neon/Supabase/RDS) over SQLite-on-a-box — not
for throughput (SQLite could serve 1000 hosts) but because multi-instance,
zero-downtime deploys and backups fall out of it. Dissolves most soft walls;
throttles/locks move into the DB.

**Stage 3 — with Stage 2 done:**
Photos → S3/R2 + CDN (upload route already returns a `src`; one-file
change) · calendar syncs as queued per-feed jobs, staggered · Sentry +
structured logs.

**Explicitly not yet:** microservices, Redis, queue infrastructure beyond
Stage 3. The shape — server actions over a scoped store — is right for this
scale; the work is the storage layer and two I/O paths.
