# Open items

Kept current — move things to DECISIONS/INCIDENTS when done, don't just
delete them.

## Deferred code findings (from the 2026-08-03 review)

- [x] **Stale-snapshot writes across `await`** — fixed 2026-08-03. All the
      reservation read-modify-write paths (upsells + verification) moved onto
      `mutateReservation` (row-locked transaction), which reads the current
      record under the lock instead of a pre-await snapshot. Covered by
      `concurrency.test.ts`.
- [ ] **Guests are never deleted** — `deleteReservation`/`deleteProperty`
      leave orphaned guest rows (name/email/phone) forever. Unbounded growth
      + a retention problem.
- [ ] **`approve/decline/markUpsellPaid` don't check the entry exists** —
      a no-op that reports success if the id is wrong.
- [ ] Dead exports: `formatDayShort`, `isRefreshing`. Duplicate
      `currentOrigin()` (billing + stay actions). Nights arithmetic in three
      places. Two different `externalIdFor` with the same name.
- [ ] Doc drift: `domain.ts` "three reasons to hold the code" — now four
      (`stay_over`).

## Scaling (full plan in SCALING.md)

- [ ] **Stage 1**: backups are now the Postgres provider's job (pick one with
      managed backups at deploy). Still open: batch the calendar cron
      (oldest-first, time budget); webhook `event.id` dedupe; reset-token digest
      lookup.
- [x] **Stage 2 (store on Postgres)** — done 2026-08-03. Store queries
      Postgres scoped by account; demo materialised; async throughout; 14 suites
      green against a test database. Provider (Neon vs Render PG) still chosen at
      deploy. Follow-ups: `scripts/apply-private-property.mjs` (seed:mine) and
      `check-guest-secrets.mjs` still read the retired `.data/overlay.json` —
      rework for Postgres or retire. Push per-account filtering (status/date)
      into SQL where hot. Move throttle/sync locks into the DB (multi-instance).
- [ ] **Stage 3**: photos → object storage + CDN; queued calendar jobs;
      Sentry + structured logs.

## Deploy checklist (updated 2026-08-03 — app is live on Render's temp URL)

- [x] `ENTRIO_SITE_URL` set (temporary Render URL) — flip to
      https://entrio.ca at DNS cutover, below.
- [ ] **entrio.ca DNS cutover** (in progress; DNS now lives on Cloudflare,
      moved off GoDaddy): custom domain on the Render service, records in
      Cloudflare, then `ENTRIO_SITE_URL` + both Stripe webhook URLs →
      entrio.ca.
- [x] Demo data: superseded — the demo is deliberately ON in production as a
      starter template (DECISIONS 2026-08-03); confirmed on the live
      dashboard.
- [x] `ENTRIO_CRON_SECRET` set + Render cron hitting `/api/calendars/sync`
      every 15 min — first run 200.
- [x] Real Stripe webhook endpoints + signing secrets — **two** event
      destinations (platform + connected), comma-separated secrets. Test
      mode; live-mode versions at the Stripe cutover.
- [ ] **Stripe test → live** (after DNS cutover): live keys, live webhook
      destinations, live Connect/Identity KYC.
- [x] Uploads on R2 with per-account keys, served via the photos domain.
- [x] Backups: Render Postgres manages them.
- [ ] Re-enable CI triggers + set Render to deploy only after CI passes
      (deliberately parked to save Actions quota — DECISIONS 2026-08-03).
- [ ] Uptime monitor on `/api/health`.

## Product backlog / not built

- [ ] Hostaway against a live API (blocked: user has no credentials; PM
      holds the account).
- [ ] Hospitable real integration (needs OAuth2 + registered client).
- [ ] Automations / messaging — **cut on purpose** (DECISIONS 2026-08-01);
      do not reintroduce without being asked.
- [ ] A live unverified test account exists in local data — offered to
      remove it, no answer yet. (Details in local memory, not here — public
      repo.)

## Housekeeping

- [ ] Keep this brain updated every session that changes architecture,
      scope, or key behaviour (protocol in README).

## From the 2026-08-06 eight-persona walkthrough (deferred, not forgotten)
- ~~Unsaved-changes guard in the property editor~~ — solved the better way
  same day: sections autosave (debounced, serialized, flushed on
  navigation/tab-hide, retry on failure). No unsaved state left to guard.
- Currency choice for manual bookings — CAD is hardcoded today.
- More than one photograph per property (the luxury-host ask).
- Team access: a second login per account for co-hosts/managers.
- Trial access to identity checks: registration says "decide afterwards"
  but the flagship Plus feature can't be tried. Business call — each check
  has a real per-use cost.
- Exports for property managers: CSV of bookings/extras revenue, outbound
  iCal of Entrio's own calendar.
- Hostaway as a first-class pickable channel once API access lands; the
  interim advice (paste per-listing .ics) now at least appears on the
  locked card.
- Airbnb listing import (considered 2026-08-06, deferred): no public API and
  bot-protected pages make URL-scraping fragile and ToS-hostile; the honest
  version is paste-your-listing + AI drafting into the editor fields with a
  found/not-found receipt (~5-9¢ per attempt on Sonnet, capped input, usage
  logged). Deferred because select-all is clumsy on phones and the paste
  ceremony undercuts the "super easy" promise. If revived: desktop-badged
  step, accept partial pastes, attempt URL fetch first with paste fallback.
  Researched further same day: managed scraper actors (Apify) take a single
  listing URL and return structured JSON (description, amenities, rules,
  times, photo URLs) for ~$2/1000 — the vendor absorbs Airbnb's markup
  churn, making URL-first viable with paste as fallback; open-source
  pyairbnb (active, internal-API-based, Python) is the free option where we
  own the breakage. ~1¢ scrape + ~5-9¢ drafting per onboarding. Still
  deferred by choice, not feasibility.

## Competitive gaps (Touch Stay review, 2026-08-07)
- Multiple languages for guest-written content — wanted, deferred by the
  host to "later on". Note it's translation of prose, not UI strings.
- AI-drafted guidebook sections (same pattern as the Nearby drafter):
  host types a few notes, Entrio drafts "Inside the flat" to rewrite.
- Deliberately refused, and should stay refused: AI chat agent and a
  message hub. Entrio's promise is that nothing reaches a guest except the
  link the host paste themselves; a bot answering in the host's voice
  breaks the one thing sceptical hosts trust it for.
- Rejected outright: QR codes (manual work in the flat), logo upload.
