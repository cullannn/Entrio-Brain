# Architecture

*Last updated: 2026-08-03*

## Stack

Next.js 16.2.12 (App Router, Turbopack) · React 19 · TypeScript · Tailwind v4
· `node:sqlite` for persistence · no ORM, no test framework, no state library.

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
src/lib/overlay-db.ts SQLite persistence behind the store
src/integrations/    seed, types, ical, places, nearby-ai, hostaway
src/payments/        Stripe: subscriptions, Connect, Checkout, Identity
src/emails/          one shell template + messages, inline-styled tables
```

## The store contract — the most important seam

Every function in `store.ts` takes `accountId` as its **first parameter** and
every row is keyed `${accountId}:${id}`. Nothing in the app reaches around
this. Consequence: swapping the storage engine touches one layer — proven on
2026-08-03 when JSON-file persistence became SQLite without any app-code
changes. The planned Postgres migration (SCALING.md) rides the same seam.

### Data model: seed + overlay replay

- `src/integrations/seed.ts` ships two **fictional** properties (The Adelaide,
  The Yorkville). The seed regenerates relative to *today* on every boot so
  the demo never goes stale.
- Everything a user changes lives in an **overlay**: new rows, patches keyed
  `accountId:id`, and tombstone id-sets. The overlay replays over the seed at
  build time.
- Sample data is per-account: each owner in `sampleOwners` gets the seed rows
  stamped with their id, with portal tokens **derived** per account (SHA-256)
  so two hosts never share a guest link.
- Demo data is **off in production** (`demoDataEnabled()`), on in development.
- Persistence: SQLite (`.data/overlay.db`, WAL). One `entries` table storing
  the overlay's own shape (record / rows / ids kinds). Writes are diffed **by
  object identity** in `overlay-db.ts` — the store never mutates a stored
  object in place, so unchanged rows compare by reference and cost no SQL.
  A one-time migration imports a legacy `overlay.json` and renames it
  `.migrated`.

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
