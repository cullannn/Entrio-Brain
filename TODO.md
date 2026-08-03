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

## Deploy checklist (before entrio.ca goes live)

- [ ] `ENTRIO_SITE_URL` → https://entrio.ca (still localhost; reset/verify
      links break otherwise).
- [ ] entrio.ca A record still parked — point it at the deployment.
- [ ] Confirm `ENTRIO_DEMO_DATA` is off (defaults off when
      `NODE_ENV=production`, but verify).
- [ ] `ENTRIO_CRON_SECRET` set + a scheduler hitting `/api/calendars/sync`.
- [ ] Real Stripe webhook endpoint + signing secret (not the CLI's).
- [ ] Persistent disk or object storage for `public/photos/uploads`.
- [ ] Backups running (Stage 1) before the first real host.

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
