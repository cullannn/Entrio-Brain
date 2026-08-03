# Security model

*Last updated: 2026-08-03*

## The promise

Arrival details (entry code, wi-fi, unit number, lockbox location) appear
when the host decides and not before, and disappear when the stay ends.
Everything else is in service of that.

## Tenancy

- Every store function takes `accountId` first; rows are keyed
  `${accountId}:${id}`; `owns()` guards every mutation.
- Server actions are **public HTTP endpoints**. Every host action resolves
  the session first and passes *that* account id to the store — a request
  carrying someone else's row id changes nothing rather than changing their
  row. Client-supplied `accountId` and `id` are stripped from patch payloads;
  client-supplied property lists are filtered to owned ids.
- Sample data is per-account with **derived** portal tokens
  (SHA-256(account, seedToken)), so two hosts holding the demo never share a
  guest link.
- Adversarially reviewed 2026-08-03 (independent model pass): "I tried to
  break tenancy and could not."

## The guest token

The `/stay/[token]` token is the whole credential: unguessable, granting
exactly one reservation, no enumeration path. Guest-side actions
(`requestUpsell`, checkout, verification) are scoped by
`requireReservation(token)` and validate against the store, never the page
(capacity re-checked server-side; checkout refuses non-approved entries;
verification writes are allowlisted to the one field a guest may set).

## Withheld means off the wire

CSS hiding is not withholding — `curl` doesn't render CSS. Enforced at the
source for each channel:

- Entry card: real code only serialised when `gate.released`; blurred digits
  are placeholders built from `codeLength`.
- Guidebook: `restricted` sections emptied server-side; template tokens that
  name secrets substitute "Unlocks with your code" until release.
- Photos: images flagged as revealing the code swap to destroyed-resolution
  variants until release (client props are serialised into the page whether
  or not they're drawn).
- Address/tagline: written to be safe pre-release (no floor, no unit).
- Verified end-to-end by `npm run audit:guest`, which fetches real guest
  URLs like a stranger and greps the bytes — including the closed-again
  (post-checkout) state. The audit itself has failed twice
  (see INCIDENTS 2026-08-03); it now proves the page is a page and reports
  what it checked.

## The gates

`doorCodeState`: `unset` / `verification` / `too_early` / `released` /
`stay_over`. Verification blocks only when mode is `mandatory` **and** the
plan is entitled — on Basic the gate collapses *open* so a guest is never
stranded behind a step nobody can run. That collapse composing with
release-on-booking caused a real leak (INCIDENTS); `nothingWithholdsCode()`
now surfaces the combination to the host on the property page.

## Auth

- scrypt hashes; HMAC-signed stateless session cookie (30 d, HttpOnly).
- Signup OTP: 6 digits via `randomInt`, 10-minute expiry, 5 attempts,
  constant-time compare. Unverified accounts can't enter the app
  (`/verify` wall, escapable to fix a typo'd address).
- Reset tokens: 32 random bytes, stored as SHA-256 digest, 1 h expiry.
  Links built from `ENTRIO_SITE_URL` config — never the Host header
  (reset-poisoning).
- Sign-in throttling: sliding window per address+caller (per-process — a
  known multi-instance gap, see SCALING.md).
- Email enumeration: reset requests answer identically either way; the
  verification email's copy is honest about what an unused signup means.

## Payments

Stripe Connect — money lands in the host's own Stripe account; Entrio holds
no funds. Checkout only for `approved` entries. Identity documents are
handled by Stripe entirely; Entrio stores verdict flags and a session id,
never names/addresses read from documents (mismatch is an enum, surfaced to
the host as judgement, never a block). Webhook verifies signatures; cron
endpoint 404s (not 401s) without its secret, so it isn't discoverable.

## Enforced by tests

`tenant-isolation` (A can't touch B, token stability, per-owner sample) ·
`actions-auth` (every exported action shows an auth guard) ·
`id-check-callers` (raw verification-flag reads forbidden outside an
allowlist) · `door-gate` (the full entitlement × mode × clock matrix, both
edges of the release window, timezone boundaries, gated template tokens) ·
`audit:guest` (the wire) · `audit:bundle` (no server secret in client
chunks) · `check-theme-contrast` (a11y).

## Known accepted gaps

Per-process throttle and sync locks (fine single-instance) · no webhook
event-id dedupe (handlers are idempotent flag-sets today) · sessions have no
revocation list (30-day expiry only) · guests are never deleted (retention —
in TODO).
