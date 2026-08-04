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

### 2026-08-03 — A verified ID that doesn't match the booking flags, doesn't hold the code
Stripe Identity can come back *verified* with a name that isn't the one on the
booking (`identityMatchesBooking: "no"`) — the `mismatch` state, shown to the host
as an "Other name" chip. The mandatory requirement still counts as met, so the
entry code releases on its normal time gate; the mismatch is the host's cue to
look, not a second lock. Confirmed deliberate: hosts want an ID on file, a
surname match is advisory, and the door shouldn't wait on our judgement of one.
(A real guest verifying their own ID matches; the mismatch is the norm only in
test mode, where Stripe's test document carries a fixed name.)

### 2026-08-03 — Reservations lead with "Now & next", not one flat upcoming list
The first tab pairs what's live now with each property's single *nearest* arrival
— the "and next" a host actually glances for. Every other future stay falls to a
separate Future tab, so the opening screen stays one-line-per-property instead of
a forward backlog. Then Past, then All. The nearest-per-property set is computed
over the same rows the list is drawn from, so filtering to one property narrows
it too.

### 2026-08-03 — Step 2 done: host photos live on R2
Uploads move to Cloudflare R2 behind a `storage()` provider shaped exactly like
`payments()`/`billing()` — R2 when all five `R2_*` vars are set, the local
`public/` folder otherwise, never a stack trace for a half-set-up deploy. Only
**host uploads** go to R2; seed photos stay bundled static assets, because the
container wipes `public/` on redeploy and that's the only durability gap. Objects
are keyed **per account** (`uploads/<accountId>/<file>`) so one host's photos are
listable and clearable on their own. **Not the AWS S3 SDK** — `aws4fetch` (~2KB)
signs and `node:https` sends (megabytes of SDK for one PUT wasn't worth it, and
the SDK's newer integrity checksums fight R2). R2 over S3 for zero egress: every
guest opening a property page pulls images, and that traffic is free. Serving is
free too — `Photo` is a plain `<img>`, so an absolute `pub-….r2.dev` URL and the
old `/photos/*` paths both render unchanged. Verified against a live bucket end
to end. Next: containerize (step 3), then deploy (step 4).

### 2026-08-03 — Step 3 done: containerized for Render
`output: "standalone"` + a multi-stage Dockerfile (node:22-slim, non-root) that
ships only the traced build — no `npm install` at run time. A `/api/health` route
pings Postgres so the platform drops an instance that can't reach the DB. The one
real subtlety: **`NEXT_PUBLIC_*` is inlined at build time**, so the Stripe
publishable key is a Docker *build arg*, not a runtime var — the only place the
"one image, many environments" ideal leaks, and fine here with a single prod
environment. Deliberately a Dockerfile over Render's native Node build: it's the
portable artifact, which is the AWS escape hatch SCALING/DEPLOYMENT already lean
on. Verified by booting the standalone server directly (Docker isn't installed
locally) — health, pages, and static assets all serve. Only step 4 (create the
Render service + managed Postgres, wire env/DNS/Stripe webhook/cron) is left, and
it's dashboard work, not code.

### 2026-08-03 — Error visibility is email-on-error, not Sentry (yet)
Going live needs to be *told* when a flow throws — but Sentry is another vendor and
a heavy dependency, against this codebase's grain and a solo operator's already-long
list of dashboards. So: Next's `onRequestError` hook logs every uncaught server
error (Render keeps the full stack) and emails a concise alert through the Resend
already wired, gated on `ENTRIO_ALERT_EMAIL`. No dashboard, no aggregation, no new
vendor or dep. Repeats of one signature are held 30 minutes so a broken flow can't
flood the inbox, and the PII-carrying stack stays in the logs, out of the mail.
Trade up to a real tracker when volume asks; the hook is the seam to do it at.

### 2026-08-03 — Step 4 done: live on Render, every leg verified in production
The full stack is up and smoke-tested on the service's temporary URL: Docker
web service + managed Postgres in one region (private networking), R2 serving
through its custom photos domain, Resend sending from the real domain, the
calendar cron every 15 minutes (five-minute polling was declined — iCal feeds
don't move that fast and it triples the load for nothing), error-alert email
armed. Stripe's current dashboard forces **two event destinations** — platform
events and connected-account events, each with its own signing secret — so the
webhook route now accepts a comma-separated `STRIPE_WEBHOOK_SECRET` and tries
each until one verifies. Production smoke test passed every leg: register+OTP,
subscription checkout, Connect onboarding, guest upsell request→approve→pay,
identity verification, R2 upload, both webhook destinations delivering — after
the 0.0.0.0-redirect incident below was fixed live. **CI stays parked on
manual dispatch** until cutover, to spend zero Actions quota during the
pre-production churn; re-enabling the triggers and gating Render deploys on CI
is part of going production-ready. Remaining: point entrio.ca at the service,
then Stripe test→live.

### 2026-08-03 — Postgres provider: Render Postgres
The provider deferred "to connection time" at cloud-architecture time is settled:
**Render Postgres**, same platform as the app — private networking, one vendor, one
bill, the lean pick at 10 hosts. Neon (serverless, branching for a throwaway staging
DB) was the alternative, rejected only to avoid a second vendor now; the store is
standard SQL, so switching later is a connection string, not a migration.

### 2026-08-04 — The first production feedback round, and the shapes it set
A day of the operator using the live product as a stranger would, folded back
in one pass. The decisions that will outlive the diffs:
**Onboarding is channel-first, then the trial.** A new host's first screen asks
where their bookings live — nothing preselected, because the answer shapes
everything downstream and a default isn't an answer — then explains the trial
(Basic, free, no card) and names Plus before it's ever needed. Once per
account, keyed on the account so it survives devices.
**Unbuilt integrations are visible but locked.** Hostaway/Hospitable show as
"Coming soon" rather than vanishing — a channel-manager host deciding whether
Entrio fits needs to see the roadmap — and the lock is enforced in the server
action, not just the card.
**The plan gate is now visible at the point of temptation.** Trial/Basic can't
*select* identity verification any more: the picker shows "Don't ask" checked
— the truth of the door — locks the asking options with the plan named as the
reason, and keeps the saved setting for the day the plan changes. The
enforcement always existed; what changed is that the UI stopped offering a
choice the plan couldn't honour.
**Extras are one-time or repeatable.** A mid-stay clean on a three-week stay
is wanted weekly; a late checkout can only happen once. Repeatable re-offers
when the last ask settles (paid or declined) and never allows two open lines —
that invariant is what keeps every upsellId-keyed mutation unambiguous, and
the store patches only the open line so paid history is untouchable.
**Preview's theme selector is a costume.** It rides a query parameter and
never writes the property; committing to a theme stays the editor's job, so
browsing six looks can't end with the wrong one live. Preview also opens
in-tab now — its bar carries the way back.
**Manual entry validates early and twice.** Property unchosen until chosen,
submit greyed until the required fields exist, emails must look like emails,
overlapping dates on one property are a refused double-booking (half-open, so
same-day turnover stays legal) — each rule in the form for the human and in
the action for the wire.
**Dev conveniences are NODE_ENV-gated, never flag-gated.** OTP 000000 and the
one-click payments stamp exist only under `next dev`; production builds can't
reach them by any configuration.

### 2026-08-04 — Plans got teeth: a property ceiling, and extras that wait for Stripe
Basic (and the trial with it) includes up to 3 properties; Plus is unlimited.
One domain function gates every path a property appears by — the add button
(greyed at the ceiling, reason printed), duplicate, and listing sync — and
our own plan switch refuses a downgrade while the portfolio doesn't fit. The
one path that can't be intercepted (a change made in Stripe's own billing
portal) freezes growth instead of shrinking anything: billing never deletes
work. And priced extras are withheld from real guests until the host's
connected account can take a charge — offer, request and checkout read one
predicate — because an extra a guest can request but never settle is a broken
promise wearing a price tag. Free extras pass, the host's own preview shows
everything, and the Upsells screen says what's hidden with the connect button
beside it.

### 2026-08-04 — The guest introduces themselves fully, and the house acknowledges it
The intro screen asks every guest for whatever their booking arrived without —
full name, email, phone, all required and live-validated — and shows what it
already knows instead of asking again: nothing known is rewritable from a
link (the claim fills blanks field by field, refuses the rest). When the
introduction lands, a staged arrival sequence — prepared, personalized,
perfected — plays before the portal reveals. The claim action deliberately
does not revalidate: any revalidation refetches the route the moment the
action returns, which killed the ceremony at half a second before the
sequence was given ownership of its own refresh. Hosts edit guests behind an
explicit Save now, not on blur. And Reservations gained a Calendar tab — two
months per property, nights as continuous blocks, clashes in clay, every
night a link that keeps the calendar underneath.

### 2026-08-04 — Perceived speed is a feature: skeletons and press states
Every host page renders dynamically, so a nav tap used to do nothing until
Postgres answered — a dead half-second that read as the app ignoring the
touch. Convention now: one loading.tsx paints the destination's silhouette
instantly for the whole host side (Settings shares it), and interactive
chrome acknowledges the press itself (active: states — the house already
preferred presses to hovers). Companion pattern for responsive tables: below
lg a row is a composed card; at lg the wrappers dissolve with lg:contents and
lg:order-* deals the same cells into the table's columns, so one markup
serves both without a parallel mobile tree.
