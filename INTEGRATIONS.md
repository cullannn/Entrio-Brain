# Integrations

*Last updated: 2026-08-03*

## Stripe

Three surfaces, one account per host (Connect):
- **Subscriptions** — Basic $29 / Plus $49, Checkout + a customer portal.
  Two price ids (`ENTRIO_STRIPE_PRICE_ID`, `…_PLUS`). Plan gates identity
  checks.
- **Connect** — guests pay for extras into the **host's** Stripe account;
  Entrio holds no funds. Checkout only for `approved` upsells.
- **Identity** — the real ID check. Documents never touch Entrio; a webhook
  writes `idVerified` / `identityFailed` / `identityMatchesBooking` (+ the
  session id, kept so a guest's return can be matched). Mismatch is surfaced
  to the host as judgement, never an automatic block.

Webhook verifies signatures. **No event-id dedupe yet** (handlers are
idempotent flag-sets; add before high volume — TODO). Local testing needs
`stripe listen` (prints a fresh signing secret each run; it's the user's
process, not the app's).

## Google Places (New) + Anthropic — the Nearby feature

Hybrid by design (see DECISIONS 2026-08-02): **Places supplies facts, Haiku
supplies judgement.** `searchText` / `searchNearby` with a tight
`X-Goog-FieldMask` (the mask is the price list — `rating` lifts Pro→Enterprise
SKU). Haiku picks from the Google-supplied list by id and writes the blurb +
host note. ≈ $0.028 / 30-place draft, and every place verifiably exists.
Category → Google-type map keyed on stable ids (`morning`, `eat`, `drink`,
`do`, `shop`). Exclusion list drops hotels/apartments; `businessStatus`
filters the closed. Blurbs may not restate ratings; policy claims banned.
Key: `GOOGLE_MAPS_API_KEY` (both Places APIs enabled).

## Anthropic (models)

Haiku for Nearby curation. Structured output via forced tool calls;
`pause_turn` continuation and `max_tokens` sizing handled (web-search path
needed a bigger budget than a 6-place ask first assumed). `ANTHROPIC_API_KEY`.

## iCal (bookings, the working path)

Per-property published `.ics`. Parser handles Airbnb / Vrbo / Booking.com /
manager wording, folded lines, LF-only endings, owner blocks vs guest
bookings, DTEND-exclusive checkout math, malformed input. **Feeds carry no
names** — guests self-introduce. Refresh: opportunistic in `after()` on host
page loads when >30 min stale; `/api/calendars/sync` for a cron (batching
needed at scale — SCALING.md). Airbnb link is what the user actually uses.

## Channel managers (Hostaway / Hospitable)

- **Hostaway** — adapter built (`listings → properties`, reservations, sync
  = append/update only). ⚠️ **Never run against live API** — the user has no
  API key yet (his PM holds the account); verified only against a stubbed
  `fetch`. Do not claim end-to-end. The `.ics` he *does* have names nobody
  ("by Hostaway" — 32 events, 2 distinct summaries; carries party size +
  channel in DESCRIPTION).
- **Hospitable** — deliberate stub. Personal access tokens aren't for
  third-party use, so a real integration needs OAuth2 against a registered
  client; self-serve registration unconfirmed.

## Resend (email)

Password reset + signup OTP. One shell template (inline-styled tables,
Cormorant/Georgia display + system sans, matches the site). `sendEmail`
never throws (a failed send must not 500 an action or leak that an address
exists) and logs to console when no key is set, so the whole flow is testable
on a laptop with no domain. Links come from `ENTRIO_SITE_URL`, never the Host
header. Domain: entrio.ca on GoDaddy (SPF/DKIM/DMARC; GoDaddy's `_spfm`
rewriting was the verification blocker).

## Cloudflare Email Routing (inbound mail)

Receive-only, and there are no mailboxes: Email Routing takes everything for
the domain, a catch-all rule hands it to the `email-rsvp` Email Worker, and
the worker splits it — `rsvp+…` replies are POSTed to `/api/rsvp` (bearer
`RSVP_SECRET`, which must equal Render's `ENTRIO_RSVP_SECRET`), everything
else is forwarded to a verified destination inbox. So `support@`, `privacy@`,
`info@` and any address anybody invents all work with no configuration.

- Both silent failures live here: an **unverified** destination routes
  nothing, and a `FORWARD_TO` that isn't set makes the worker drop non-RSVP
  mail without an error. Symptom is identical — mail vanishes.
- A secret mismatch between the worker and Render breaks only the cleaner
  RSVP path, and breaks it quietly: the invite sends, the reply arrives, the
  turnover never moves.
- Catch-all also forwards spam to invented addresses. If it ever gets noisy,
  swap to explicit rules — but keep one for `rsvp+*` or the RSVP path dies.
- **Sending is a different system** (Resend). Replying *as* an @entrio.ca
  address needs an SMTP relay in the mail client's "send mail as"; Email
  Routing cannot send.

Full wire-up in the app repo's `DEPLOY.md`.

## Environment variables

`RESEND_API_KEY`, `ENTRIO_MAIL_FROM`, `ENTRIO_SITE_URL`,
`STRIPE_SECRET_KEY`, `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY`,
`STRIPE_WEBHOOK_SECRET`, `ENTRIO_STRIPE_PRICE_ID`, `…_PLUS`,
`GOOGLE_MAPS_API_KEY`, `ANTHROPIC_API_KEY`, `ENTRIO_CRON_SECRET`,
`ENTRIO_DATA_FILE` (throwaway data target), `ENTRIO_DEMO_DATA`,
`ENTRIO_ACCOUNT` (audit target).
