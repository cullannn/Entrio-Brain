# Architecture

*Last updated: 2026-08-03*

## Stack

Next.js 16.2.12 (App Router, Turbopack) · React 19 · TypeScript · Tailwind v4
· Postgres via `pg` (pooled) · no ORM, no test framework, no state library.

⚠️ This Next.js version has breaking changes vs. public docs. The main repo's
`AGENTS.md` says it plainly: read the guides in `node_modules/next/dist/docs/`
before asserting anything about framework APIs.

## The three surfaces

1. **Guest portal** — `/stay/[token]`. No login; the unguessable token *is* the
   credential and grants exactly one reservation. A four-tab phone app
   (Stay · Arrival · Guide · Nearby), not a scrolling page — see DECISIONS
   2026-07-31.
2. **Host app** — `/host/*`. Session-cookie auth. Overview dashboard,
   Reservations, Upsells, Properties (each property edited on one page with
   tabs: Photograph, Basics, Calendar, Pre-arrival, Arrival, Guidebook, House
   rules, Nearby, Theme), plus `/settings` (Plan, Channels, Payments, Features,
   Account).
3. **Marketing / auth** — `/`, `/register`, `/signin`, `/verify`, reset flow.

## Layers

```
src/app/…            routes + server actions (actions are PUBLIC endpoints)
src/components/…     host/ and portal/ component trees, ui.tsx primitives
src/lib/domain.ts    pure business rules — no I/O, fully unit-tested
src/lib/store.ts     THE data contract (see below)
src/lib/db.ts       Postgres pool + schema behind the store
src/integrations/    seed, types, ical, places, nearby-ai, hostaway
src/payments/        Stripe: subscriptions, Connect, Checkout, Identity
src/emails/          one shell template + messages, inline-styled tables
```

## The store contract — the most important seam

Every function in `store.ts` takes `accountId` as its **first parameter** and
every row is keyed `${accountId}:${id}`. Nothing in the app reaches around
this. Consequence: swapping the storage engine touches one layer — proven on
JSON-file → SQLite → Postgres, each without app-code changes — the seam held.

### Data model: normalized rows, demo materialised

- `src/integrations/seed.ts` ships two **fictional** properties (The Adelaide,
  The Yorkville) as a fixture. Claiming the demo inserts them as real rows.
- Sample data is per-account: each owner gets the seed rows stamped with their
  id, with portal tokens **derived** per account (SHA-256) so two hosts never
  share a guest link.
- Demo data is **off in production** (`demoDataEnabled()`), on in development —
  so production is purely a host's own rows.
- Persistence: **Postgres** (`src/lib/db.ts`, pooled `pg`), one table per
  entity — `jsonb data` plus the columns anything queries by (`account_id`,
  `id`, `portal_token`, `external_id`). Patches are `data || $x::jsonb` (shallow
  merge, matching the old object spread). The store is **async**; `domain.ts`
  is pure so the ripple stayed in pages/actions. Provider-agnostic standard SQL.
- The demo is **materialised, not replayed**: claiming the sample inserts real
  stamped rows once. No tombstones/patches/regeneration — a delete is a DELETE.
  (History: was SQLite, and before that a whole-file JSON overlay. See
  DECISIONS 2026-08-03.)

## Key flows

### Door-code gate (`domain.ts: doorCodeState`)
The product's core promise. Five states:
`unset` → `verification` → `too_early` → `released` → `stay_over`.
- Release: property's `releaseLeadMinutes` before check-in, in the
  **property's timezone** (`RELEASE_ON_BOOKING = -1` releases immediately).
- Close: midnight at the end of checkout day, property timezone — late enough
  for the forgotten-charger run, closed by the next morning.
- Verification only blocks when the mode is `mandatory` **and** the host's
  plan is entitled (see SECURITY.md — the entitlement collapse is deliberate).
- Everything "withheld" is withheld **from the wire**, not hidden by CSS —
  including guidebook template tokens and locked photo variants.

### Identity checks (Plus feature)
Property modes `off | optional | mandatory`; Stripe Identity does the real
check; a webhook writes the verdict; a host can override either way.
`idCheckState()` in domain.ts is the single source for "where is this guest's
check" — six states (verified / mismatch / failed / outstanding / optional /
exempt), and a lint-like test forbids reading the raw `idVerified` flag
outside an allowlist (three bugs came from exactly that).

### Upsells (approve-then-pay)
`requested → approved → paid`, `declined` as the other exit. **Nothing is
charged before the host approves** — `startUpsellCheckout` refuses anything
not `approved`, so the gate holds even against direct action calls. Capacity
is re-checked at request time and again at approval. Money lands in the
host's own Stripe account (Connect); Entrio never holds funds.

### Bookings
- Airbnb iCal per property: paste a link, it refreshes on host page loads via
  `after()` when >30 min stale; `/api/calendars/sync` exists for a cron
  (404s without `ENTRIO_CRON_SECRET`). Feeds carry **no guest names** by
  design — guests introduce themselves on first portal open (`WhoAreYou`).
- Sync law: imports **append and update by externalId, never overwrite** host
  work. Cancelled bookings are kept and marked, never deleted.
- Manual entry for direct bookings; Hostaway adapter built but never run
  against live credentials; Hospitable deliberately a stub (see
  INTEGRATIONS.md).

### Auth
scrypt password hashes · HMAC-signed stateless session cookie
(`accountId.issuedAt.sig`, 30 days, HttpOnly) · 6-digit OTP email
verification at signup (10 min, 5 attempts, constant-time compare) · reset
tokens stored as SHA-256 digests (1 h) · per-process sign-in throttle.
`requireAccount` → any signed-in account; `requireActiveAccount` → also
refuses lapsed subscriptions (writes blocked, reads allowed) and unverified
email (redirects to `/verify`).

## Testing (see CONVENTIONS.md for the rules)

14 plain-Node suites under `tests/`, no framework, run by `tests/run.mjs`.
Two wire-level audits that need a running server: `audit:guest` (withheld
secrets never reach the response body — checks real HTTP against derived
guest links) and `audit:bundle` (no server secret in client chunks). Browser
verification happens at real viewports; 414 px via same-origin iframe.
