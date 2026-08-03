# Decision log

Chronological. Append new entries at the bottom with a date and a *why* —
the what is in git.

---

### 2026-07-30 — The project exists
A luxury replica of Enso Connect's guest-experience platform, both halves
(guest portal + host dashboard) in v1. Custom design system: Cormorant
Garamond + Inter, ink/bone/brass palette. Bookings behind a swappable
adapter because Airbnb has no public API.

### 2026-07-31 — The portal is a phone app, not a page
First build was an editorial long-scroll; rejected ("massive wall of text").
Now a four-tab shell (Stay · Arrival · Guide · Nearby), fixed bottom tab bar,
guidebook accordions default closed, Extras deliberately on the Stay screen
because it converts on sight. **Any portal change gets checked at 414 px.**

### 2026-07-31 — Interaction language
Recessed = information, raised = tappable, filled = the primary action.
Press states over hover states. The entry-code card is brass-wash, never
dark. Affordances match physical reality: no copy button for a mechanical
lockbox's wheels, no copy button for a wi-fi *network name*.

### 2026-08-01 — Scope cut: no chat, no automations
Removed from the codebase, not hidden. The product is: check-in link per
guest, guidebook, approve-then-pay extras. **The approval gate is the
product** — nothing is charged before the host says yes, so there is never
anything to refund.

### 2026-08-01 — Nothing real in the seed
The seed ships to every install, so it holds only fictional properties. The
host's real property lives in gitignored local files and is patched over the
sample locally (`npm run seed:mine`). Git history was scrubbed before the
first push because the seed had briefly held real details.

### 2026-08-02 — First push to GitHub (private repo), 55 commits by 08-03
Commit style: prose-first messages that carry the why; the brain repo now
holds the cross-commit narrative.

### 2026-08-02 — Guests name themselves
Calendar feeds carry no names (Airbnb writes "Reserved", Hostaway "by
Hostaway"). Instead of importing junk, the portal asks the guest who they
are on first open. A booking reads "Awaiting guest" until then.

### 2026-08-02 — Channel sync is append-only, by law
A sync may create and update *its own* rows (matched by externalId) and must
never overwrite host-authored work. Cancelled bookings are kept and flagged,
never deleted — the host may have attached codes, guests, or paid extras.

### 2026-08-02 — Three channel states, not two
"I'll add bookings myself" / "Direct through Airbnb" (iCal per property) /
channel manager (Hostaway/Hospitable). The middle state exists because it is
the situation most independent hosts are actually in. New accounts default
to Airbnb-direct (2026-08-03), and on that setting a property without a
calendar link is **not ready** — the page would never see a guest.

### 2026-08-02 — Nearby runs on Places + Haiku, not a big model
Measured: model-only $0.03/draft but hallucinated places; model + web search
$1.50 and timed out; Google Places for facts + Haiku for judgement ≈ $0.028
and every place verifiably exists. Field mask is treated as a price list
(`rating` lifts the SKU tier). Blurbs must say something a rating can't;
policy claims ("book ahead") are banned after one contradicted a host note.
Five default categories follow a day: Morning, Dining, Drinks, Things to do,
Shopping. Category ids are stable (`morning`, `eat`…) so renames never
orphan places.

### 2026-08-03 — Plans: Basic $29 / Plus $49; identity checks are Plus
On Basic, a property set to require ID **collapses the verification gate**
rather than stranding a guest behind a step nobody can run. That collapse is
correct and also composes dangerously — see INCIDENTS (entry-code leak).
Consequence encoded: every read of verification state goes through
`idCheckState()` / `idCheckMode()` with the entitlement; a test forbids raw
flag reads.

### 2026-08-03 — Auth hardening round
OTP email verification at signup; password reset via Resend; emails match
the site's look (inline-styled tables, Cormorant/Georgia stack). Reset links
built from config, never the Host header. The verify screen is escapable
(change address / sign out) so a typo'd email can't brick an account.

### 2026-08-03 — The door closes as well as opens
`doorCodeState` gained `stay_over`: the code, wi-fi and lockbox note close at
midnight on checkout day (property timezone). Before this, every past guest
kept a live page with the current code forever. Midnight, not checkout time,
so a same-day "forgot my charger" return still works.

### 2026-08-03 — Preview always shows both sides of the gate
"Preview as a guest" always opens the fabricated sample stay (which can flip
before-arrival / after-check-in), never a real booking's page. Real pages
are one click away under Reservations.

### 2026-08-03 — Storage moved to SQLite (`node:sqlite`, no new dependency)
Kept the overlay's shape rather than normalising — the seed-replay model is
what keeps the demo alive, so the patch set is the thing worth persisting.
Writes are diffed by object identity in one place instead of teaching 19
call sites what they touched. Migration from JSON is one-time and keeps the
old file as `.migrated`. This is explicitly a waypoint: the Postgres move
(SCALING.md) replaces the in-memory world model, not just the file.

### 2026-08-03 — Scaling direction agreed
Target 1000+ hosts. Stage 1: backups, cron batching, webhook dedupe.
Stage 2: managed Postgres behind the existing store contract, normalised
tables, indexed scoped queries. Stage 3: photos to object storage + CDN,
queued calendar jobs, error tracking. Explicitly rejected for now:
microservices, Redis, queue infra beyond Stage 3.

### 2026-08-03 — This brain exists
Public repo, so the no-real-details rule applies here with no exceptions.
Maintenance protocol in README.md; pointer added to the main repo's
AGENTS.md.

### 2026-08-03 — Cloud architecture decided (full detail in DEPLOYMENT.md)
Going live on entrio.ca. **Always-on container on Render**, not serverless —
predictable flat cost, low ops, the app is a long-lived `next start` server.
**Postgres-first, then deploy** (not a SQLite Phase-0) — the user chose to do
the storage migration once, properly, so the first deploy is multi-instance-
ready and there's no second migration. This also drops Litestream: a managed
Postgres backs itself up. **Photos → Cloudflare R2** (zero egress on image
serving, S3-compatible). **Postgres provider deferred to connection time** —
migrate provider-agnostic (standard SQL) so Neon or Render Postgres both fit.
Build order: (1) Postgres migration behind the unchanged store contract —
tenant-isolation/door-gate/audit:guest stay green as the proof; (2) photos →
R2; (3) containerize (`output: standalone` + Dockerfile); (4) deploy + wire
Postgres/R2/cron/DNS/Stripe webhook. Rejected: Vercel (usage-based cost, seat
floor, same migration up front anyway).

### 2026-08-03 — The demo is a starter template, on in production
New hosts get the demo in production too, but pared to a *template*: the two
example properties (fully built — guidebooks, rules, neighbourhood, photos) and
the extras configured against them. **No reservations** (so the dashboard shows
the host's own numbers, not fictional revenue — the original reason demo was
off in prod), **no guests** (they only exist for bookings), **no tasks** (the
seeded ones name seeded guests, nonsense without the bookings). Supersedes the
earlier "demo off in production" decision. `demoDataEnabled()` defaults on;
`ENTRIO_DEMO_DATA=0` turns it off (tests use that). A guest link now only exists
once a host makes a real booking, so the per-account seed-token derivation
(`tokenFor`) is gone.

### 2026-08-03 — Step 1 done: the store is on Postgres
The store queries Postgres scoped by account, retiring the in-memory
seed+overlay world. **One table per entity: jsonb plus the columns anything
queries by** (account_id, id, portal_token, external_id) — not column-per-field,
because nothing filters on nested fields in SQL. Patches are `data || $x::jsonb`
(shallow, matching the old spread). **The demo is materialised, not replayed** —
claiming the sample inserts real stamped rows once, which deletes the entire
tombstone/patch/regenerate machinery (it only existed because seed rows came
back each boot). Production (demo off) is purely a host's own rows. **The store
became async** (pg has no sync API); domain.ts being pure confined the ripple to
pages and actions, and a second sweep caught the floating unawaited writes tsc
can't flag — the vanished-booking channel cancel was one. `pg` is the only new
dep; provider-agnostic standard SQL so Neon or Render Postgres both fit. Tests
run against a dedicated `*test*` database, dropped/recreated per suite. Next:
photos → R2, then containerize + deploy.
