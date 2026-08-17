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

### 2026-08-04 — The homepage shows the product instead of describing it
Rebuilt photography-led in the house language: light editorial hero, an
original generated Toronto interior (generated on the operator's own account
precisely so no stock or other host's listing is ever marketed with), and a
faithful reproduction of the guest's real Stay screen as the centrepiece —
entry-code card, essentials grid, tab rail. Iterated from three rejected
directions: generic-SaaS mocks failed ("looks too simple"), fluffy copy
failed ("only the most important information"), invented mock screens failed
("show what a guest would actually truly see"). The rules that survived:
copy is one idea per section; every number renders from PLANS/TRIAL_DAYS so
the page can't disagree with billing; the mock mirrors the live portal and
must be updated when the portal changes.

### 2026-08-04 — The homepage's centrepiece is the portal itself, rotating
The guest-screen mock became a deck of the four real tabs, rendered from the
portal's own markup at true phone scale and scaled to fit — scrollable,
because the real screens are, with Stay's entry-code card ticking a live
countdown and Arrival showing the released code. Fidelity rule hardened into
convention: the mock IS the portal's markup, and a portal redesign must be
mirrored there. Interaction lessons paid for in debugging: never commit
state from inside a React updater (dev double-invocation eats or doubles
steps), and never pointer-capture a gesture that starts on a child control —
capture retargets the eventual click to the captor and the control goes dead
under a real mouse while synthetic tests pass.

### 2026-08-04 — The brand mark is a component, not an asset
The identity settled on the seal: the display-face E inside a double ring,
drawn as a React component entirely from the app's design tokens — border,
ink, and font, no image file — so it inherits any guest theme including the
dark ones for free. Favicon and apple icon are generated at build/request
time with next/og from the same vocabulary (satori needs a bundled TTF, not
woff2; the static favicon.ico is gone). The wordmark is now a lockup, seal
beside the name with the type a deliberate notch smaller than the emblem,
and every surface — homepage, host pages, guest portal — closes with the
same small stacked seal-over-name sign-off in place of a "powered by" line.
One mark, one component, themed everywhere; a new surface signs itself by
importing it, never by redrawing it.

### 2026-08-04 — The host calendar keeps hotel time
The multi-week availability strip adopted the hotel convention: a stay bar
runs from the middle of its check-in day to the middle of its checkout day
(±50%-of-a-cell margins on the grid spans), so a turnover day shows the
departing and arriving stays sharing one square instead of an ambiguous
full-day collision. Date cells carry a weekday initial above the number.
Lesson: when bars vacate half a cell the track behind them shows, so the
grid track itself must be painted the surface colour, not left to the page
background. The feeds panel also gained a prominent button to the channel's
own calendar page (the stable calendar-router path, which survives login
redirects) — hosts kept hunting for where their iCal address lives.

### 2026-08-04 — The arrival ceremony ends on the mark, and phones arrive with their code
The guest-intro ceremony no longer exhales to a blank: the staged welcome
lines crossfade into the centered seal and name, which hold a beat and then
fade before the router refresh reveals the portal — brand as the last beat
of the handoff, with the whole sequence collapsing to about a second under
reduced motion. The phone question split into a country-code picker plus a
national number, stored composed as +code number so a host's phone dials it
from any country. UI pattern worth keeping: a styled facade showing just
"+44" with an invisible native select stretched over it — the closed
control stays compact while phones keep their picker wheel and desktops
keep type-ahead, no custom dropdown to maintain. Validation judges the
number together with its code once one is chosen, and alone before, so a
fine number isn't flagged while the code is still unpicked.

### 2026-08-05 — Manual dates move; synced dates say where they live
A manually recorded stay's dates are editable from its pane, with the same
ordering and overlap rules as creating one (the stay excluded from its own
clash check) — everything attached to the stay survives, which is the point:
the old delete-and-recreate workaround rotated the portal token and killed
the link already sent to the guest. Synced stays are not editable here and
say so: the channel's feed patches their dates on every sync, so a local
edit would be silently stomped within the hour. The manual/synced split is
enforced server-side off the reservation's source, never trusted from the
pane. Source-absent is treated as manual — the safe reading for every row
that predates the field.

### 2026-08-05 — The extras conversation goes by mail, and addresses are proved
Upsell requests email the host (with a link to the stay pane) and the host's
approve/decline emails the guest when an address is on file, linking to the
portal where Pay lives — sends are never fatal, the decision stands
regardless, and the request-side send only fires when a line was actually
appended (the dedup answers a double-tap with ok, which must not mean two
emails). A guest-typed email is now proved before it lands: six-digit code
to the inbox, typed back — the sign-up-code lifecycle reused verbatim
(hashed, expiring, attempt-counted, spent on use) plus one extra pin: the
code only confirms the exact address it was minted for. Guests whose address
arrived with the booking never see the step. Test technique worth keeping:
with no mail key the send goes to the console, so tests capture console.log
and read the code out of the mail exactly the way a developer does.

### 2026-08-05 — The intro polishes its edges; the mail wears the seal
Three refinements in one pass. The OTP input auto-submits on its sixth digit
and carries the full autofill recipe (one-time-code autocomplete + numeric
pattern + name) so phone keyboards offer the code from the inbox. The email
template now opens with the seal-beside-ENTRIO lockup and closes with the
stacked sign-off — the seal drawn as nested border-radius circles, which
degrade to a bordered square in Outlook's Word engine and still read as a
mark. And the intro form shows what the booking already knows: host-supplied
fields render recessed and readOnly with a "from your booking" tag — visible
because hiding them read as ignorance, locked because a forwarded link must
never rename someone else's stay (readOnly over disabled: screen-readable,
copyable, out of the tab order).

### 2026-08-05 — The host's inbox mirrors the guest's milestones
Every pre-arrival milestone now mails the host: extra requested (already),
extra paid, guest introduced themselves, identity check passed (with the
name-match verdict). Patterns that made it safe: (1) payment mail lives in
the single settle point both the return trip and the webhook pass through,
behind the existing idempotency guard — one mail per payment however many
arrivals; (2) where two paths race to flip a milestone (guest return vs
webhook on identity), the transition is detected *inside* the locked mutate
via a closure flag, so only the first flipper mails; (3) the simulator path
mails too — a moment that mails in production must mail on a laptop, or the
flow can't be rehearsed; (4) a shared host-notify module owns the wording,
callers own the "did it just happen" decision. Sends never fatal.

### 2026-08-05 — Turnovers: an opt-in schedule where cleaners are links, not logins
A Settings switch adds a Turnovers tab (below Properties) with two inner
tabs: the schedule — every confirmed checkout as a filtered table in the
Reservations manners — and a cleaner roster. Decisions that shaped it:
cleaners never sign in; their whole interface is an invite email and a
tokened accept/decline page (the guest-link credential pattern reused). The
calendar sync is plain iTIP: a hand-built ICS (METHOD:REQUEST, stable UID
per job, counted SEQUENCE, UTC times converted from the property's zone)
attached to both the cleaner's invite and the host's copy — Google/Apple
treat re-issues as updates. Acceptance re-issues the event with the
attendee ACCEPTED and a bumped sequence so the host's calendar confirms in
place. The clean window is checkout→next check-in on same-day turnovers
(labelled a hard deadline) or checkout+4h otherwise. Answers may change;
each change mails the host exactly once (decided under the row lock). No
re-assign shortcut — withdraw (which tells the cleaner) then invite.
State lives on the reservation (`cleaning`), the roster in its own table;
the answer token gets an expression index on the jsonb. Lesson: moving a
route directory mid-session poisons Turbopack's chunk graph ("CJS module
can't be async" against the old path) — restart dev after renames.

### 2026-08-05 — The calendar's Yes flows in, and the table settled
Inbound RSVP: with ENTRIO_RSVP_INBOUND set, cleaning invites carry an
Entrio-owned organizer address plus-addressed with the job token; Google's
iTIP REPLY lands on a svix-verified Resend inbound webhook that reads
PARTSTAT (or the Accepted:/Declined: subject) and drives the same tokened
answer path — locks, dedupe, host mail. Unset, the host stays organizer.
Table lessons: per-row grids with an auto column misalign the moment rows'
controls differ (fix: fixed-width control column); mobile and desktop row
shapes are spelt out separately rather than contorted from one markup;
cards gapped in the hand, zebra rows at the desk.

### 2026-08-05 — Eight personas rewrote the homepage's rules
A persona-panel review (portfolio/PMS, single-condo novice, boutique
hospitality-lover, guest-experience skeptic, ESL reader, retired teacher,
literature stylist, 30-second skimmer) converged on principles now applied:
lead with the mechanism, not the outcome (the H1 must say what the thing
IS); answer "how do bookings get in" and "does my lockbox work" above the
fold; do the payback math on the page ("one late checkout covers Entrio");
claim outcomes the demos only demonstrate (no 11pm code-texting; expired
codes lock out past guests); never borrow another industry's idiom ("no
seats" meant furniture to most). The heading rule that survived all eight:
an inversion may keep its comma only if the noun stays ("Closed after
checkout" passes; "Entry, timed" failed — "the bodies rescue the headings,
and a heading shouldn't need rescuing"). Register map: literary voice is
licensed inside the guest phone screens and hero; pricing, plans and
mechanics must be plain. Guest-visible idioms must survive literal
translation ("the walls are honest" did not).

## 2026-08-06 — Eight personas walked the app; the app answered
Ran an eight-persona end-to-end walkthrough (first-timer, engineer,
property manager, retiree, luxury, side-hustle, ESL, sceptical veteran) as
parallel agents reading the real screens. Convergent findings got fixed the
same day: the sample portfolio no longer hides the "add your own property"
door or eats the plan ceiling; the turnover feature carries one name
(Turnovers) and its empty state deep-links to the right Settings tab; the
guest-link card explains the Airbnb-thread handoff instead of showing two
dead buttons; payment setup copy stops assuming a Stripe account and the
platform fee is disclosed before connecting; nav words now match page copy
(Extras, Bookings); the identity email no longer says "passed" on a name
mismatch; idioms trimmed for ESL readers; arrival ceremony halved.
Lesson: seed data changes which empty-states are reachable — design the
first morning WITH the sample present, not the zero state.

## 2026-08-06 — Onboarding became a product surface, audited like one
Built the guided onboarding stack: a "Getting set up" checklist on the
Overview whose ticks DERIVE from the same propertyGaps logic that writes
the property page's red line (stored progress can lie; derived progress
can't), a ten-step live tour that narrates real pages (the sample
property's tabs, Extras, Payments, Turnovers) with its stop index in the
URL so the card survives cross-page walks, and a long-form guide artifact.
Then ran the persona panel against the finished guide asking one question
— is "checklist done" the same as "live"? — and it wasn't, three ways:
payments lived outside the required path while priced extras silently
hide from guests; nothing made the host ever SEE their own guest page;
and the first-booking moment (import-on-open, "Awaiting guest", no
new-booking email) was unexplained. All three became steps or card copy.
Lessons: audit the guide with the same rigor as the product; tick
completion by witnessing (the Guest-view step marks done only when the
preview actually renders); and any step teaching a button must quote the
button's real label.

## 2026-08-08 — The landing page teaches the category before it sells the product

A close read of the market leader's homepage found the gap wasn't
features, it was **order**. They define the category ("what is a digital
guidebook?") before naming a single feature; ours went headline →
benefits, leaving a reader who has never seen this kind of product to
assemble the concept themselves. Fixed by adding, in this order: a
definitional band under the hero, and a three-step "what you do" section
covering the host's own work rather than the guest's experience.

Two structural lessons that generalise past this page:

- **The concepts a product depends on have to be taught somewhere.** The
  sign-up line said "paste your calendar link" to people who don't know
  they have one. Removing the jargon from the call to action only works
  if the idea is introduced earlier; otherwise the page has quietly
  dropped a required step.
- **A decision point without an action is a leak.** Neither pricing card
  had a button, so a reader who chose a plan had to scroll back to the
  hero. Every place the page asks someone to decide now lets them act.

Deliberately not copied: emoji feature-strips in the subhead, noun-label
feature names ("Upsell widget", "Data dashboard" — full sentences say
what happens and read better), and headline proof numbers, which we have
no honest equivalent of until real hosts are using it. Fabricated social
proof is off the table; the open question is whether "built and run on a
real rental" earns its place instead.

## 2026-08-08 — The landing page shows the host's half, in real screenshots

The page proved the guest's experience with faithful reproductions of the
guest screens and asked hosts to take *their* half on faith. The editor —
the thing a host actually spends an evening in, and the place the product
is at its most persuasive because the guest's page updates as you type —
was invisible until after sign-up. Fixed with three screenshots of the
real host app, captioned, between "how it works" and pricing.

Two things worth carrying forward:

- **Screenshots of your own dev environment are a data-exfiltration path.**
  This developer's local database has a real property applied over a
  sample id, and a real channel calendar token pasted into a sample's
  calendar field. Both looked like fiction on screen. Every field in a
  candidate shot was traced back to the seed file before shooting, and one
  screen (the calendar) was dropped when its contents turned out not to be
  in the seed. Rule: if a value in the frame can't be found in seed source,
  it doesn't ship.
- **Real screenshots buy credibility and owe maintenance.** They go stale
  when the editor's layout moves, so they belong on the list of things a
  UI change has to update. The alternative — hand-drawing the host UI in
  components, as the guest phone deck does — costs a day and can drift
  from the real thing silently, which is worse than drifting visibly.


## 2026-08-08 — The legal pages are written from the code, not from a template

`/privacy` and `/terms` went up, drafted field by field from what the software
actually does rather than adapted from a template. The reasoning is that a
template is what creates the legal exposure for a one-person business: it
promises things the product doesn't do ("we never share your data") while
Stripe, Resend, Cloudflare and a US host all hold some of it.

Consequences worth keeping:

- **No consent banner, and the page says why.** One cookie, `entrio_session`,
  set at sign-in; no analytics, pixel, session recorder or third-party tag
  anywhere. That is a real competitive line, not just compliance.
- **Retention describes what happens, not what sounds responsible.** Nothing
  deletes automatically, so the page says so. A twelve-month promise no code
  keeps is worse than no promise.
- **Prices and trial length are read from PLANS/TRIAL_DAYS**, the same objects
  Stripe bills from, so the terms cannot drift from what a card is charged.
- Cross-border storage (Render Ohio, Stripe US) is disclosed because Canadian
  law requires it, and identity documents are described as Stripe-held
  because they are — Entrio stores only the verdict.

Neither page has had a lawyer's eye. The draft is accurate, which is the part
software can be responsible for; sufficiency is not.

## 2026-08-10 — Every theme is a listing type, and every theme has a home

Themes renamed to what hosts search for on Airbnb: Coastal→Cottage,
Estate→Villa, Atelier→Studio (Loft already was one). Second rename for two
of them; ids never change, which is what makes renames free.

Four new sample properties give each theme a worked example — Cottage,
Villa, Loft, Studio — bringing the template set to six. Deliberate spread:
each demonstrates a different entry method (lockbox, fob+concierge, keypad,
gate+lockbox, freight lift, in-person key handoff), so every access path
the product supports has a template showing how to write it up. The Loft
carries the pet extra. All content in the simplified register: 4–6 arrival
steps, one instruction each, because hosts copy the template's *style*.

The lesson that generalises: **materialised seed data needs a top-up
path.** The template is inserted as real rows at account creation, so any
sample shipped later would exist only for accounts newer than it. The host
layout now inserts missing sample ids for accounts that still hold part of
the sample set (ON CONFLICT DO NOTHING, so edited samples and the local
private-property patch are untouched); an account that cleared its samples
holds none and stays cleared. Any future sample ships to everyone by
itself.

## 2026-08-11 — Entrio has its own set of keys, and they are not a host's

An operator's portal at `/admin`: every account, what they're entitled to,
where that entitlement came from, and whether they've built anything with it.

**The gate is a login of its own** — `ENTRIO_ADMIN_PASSWORD` plus
`ENTRIO_ADMIN_EMAILS` (defaulting to admin@entrio.ca), both environment
variables, deliberately *not* a flag on a host row. A column is one stray
UPDATE, one jsonb merge with a user-supplied key, or one over-broad settings
action away from a host granting themselves the run of everybody else's data;
an environment variable can only be changed by somebody who can already reach
the deploy — the person the flag would have been protecting. It also has no
bootstrap problem, and admin@entrio.ca is not a host account, so there is
nothing for a mistake there to leak sideways into. Unset password ⇒ `/admin`
is a **404, not a locked door**, which is the right answer for a fork, a
preview deploy and a laptop. The unlock ticket is a signed cookie scoped to
`/admin`, twelve hours, carrying a fingerprint of the password — so rotating
the password revokes every ticket already issued.

**Granting a plan is a separate field from paying for one.** `compTier` sits
beside `tier` rather than in it, because `tier` is written from the price on a
Stripe subscription and nothing else — a grant stored there would be erased by
the next webhook that mentioned the account. `compTier` outranks both the
trial clock and the subscription in `planOf` and `accessState`, so every
entitlement in the app honours it (property ceiling, identity checks), and it
survives a cancelled subscription and a lapsed trial. Revoking is a deletion,
not a restore: the fields underneath were never overwritten.

The portal can also push a trial back (counted from today, not from the old
date — extending an expired trial by a fortnight otherwise leaves it expired)
and **empty an account back to the day it was made**, behind the account's own
address typed out in full. That last one exists because the alternative is a
hand-typed DELETE against production at midnight. What is deliberately absent:
delete-account, sign-in-as-host, edit-their-data. Refunds and cancellations
stay in Stripe, which holds the record and the audit trail.

## 2026-08-11 — The simulator's kindness has a cost, and it needed a page

Every payment path falls back to a simulator when its configuration is
incomplete — on purpose, so a half-set-up deploy serves a guest a working page
rather than a stack trace. The cost: **"live" and "convincingly pretending"
look identical from outside.** No error, no banner — a host who has connected
their bank, a guest who has "paid", and no money anywhere.

`/admin/stripe` states the difference out loud, and asks Stripe rather than
reading our own flags: the platform account's activation, each plan's price
and which set of books it belongs to, whether Connect and Identity answer at
all, the webhook secret, the return address, both key modes agreeing. Nothing
it reports is a code change — the app is entirely key-driven, so going live is
environment variables and dashboard switches.

The one genuinely dangerous combination it exists to catch: a live secret key
beside a test publishable key. The server creates a real payment intent, the
browser is handed a client secret from the other mode, and the card form
refuses every card — with no error that names the cause.

## 2026-08-11 — Deletion asks Stripe, not us

The admin portal can delete an account outright — the login and every row filed
under it, one transaction. Three fences, each stopping a different mistake:

1. **The account's own address, typed out.** The accounts list is a table of
   similar rows; the failure worth preventing is deleting the one *next to* the
   one you meant, which is silent and immediate.
2. **Stripe's answer about whether they're still paying** — not our `status`
   field. The two disagree in both directions and only one holds the money: a
   webhook we never received leaves `status` reading trialing over a live
   subscription, and a cancellation made in Stripe's own portal leaves it
   reading active over nothing. A subscription id that doesn't *resolve* is
   not a live subscription and must not block deletion forever on the strength
   of a string nobody can look up.
3. **A refusal when Stripe can't be reached at all.** "I don't know whether
   they're paying" is a reason to stop, not a reason to proceed.

The consequence the screen states plainly, because it reaches somebody who
agreed to none of this: **every guest link on the account stops working** — a
stay resolves by token against a reservation row, and those go with the rest.
The count of affected bookings is shown before the button.

Stripe is untouched by deletion. A connected account belongs to the host;
closing it is theirs to do.

## 2026-08-13 — The trial belongs to Basic, and Basic keeps it

Checkout passed Stripe no trial parameters at all, so a host who subscribed on
day 3 of 30 was charged that afternoon and forfeited 27 days they had been
promised. The rational response is to wait until day 29 — a system that pushes
its most willing users to delay.

**Basic now carries `trial_end`.** The card is *saved* at checkout, the
subscription starts immediately in Stripe status `trialing`, and the first
invoice falls due when the free trial would have ended anyway. No money moves
that day.

**Plus and Ultimate charge immediately** and start immediately. They are bought
for what they add — identity checks, the property ceiling lifted — and someone
reaching for those on day 4 wants them that afternoon. Handing over the feature
and billing three weeks later is a free month nobody offered.

The reasoning that decided it, for when this is revisited: the card on file is
worth more than the float on $29. With one saved, day 30 is an automatic
conversion; without one it is a fresh decision made by somebody who may be
busy or lukewarm. Trial-to-paid conversion is dominated by whether the payment
method is already there, and charging on subscribe works against exactly that.
It would flip if the trial were short — at 7 days the float is trivial and
immediate charging is simpler to explain.

Stripe refuses a trial ending sooner than 48 hours out, so the last two days of
a trial are charged today rather than erroring in front of somebody trying to
pay. Both plan cards state which case they are *before* the button.

Both `reconcileSubscription` and the webhook already map Stripe `trialing` →
our `active`, so a trialing subscription keeps access with no change.

## 2026-08-13 — Rates agreed with one host, and Stripe owns the arithmetic

Custom discounts from the admin portal: percent or amount off, for one of
Stripe's three durations (once / repeating N months / forever). **A Stripe
coupon does the maths.** The discount has to survive an upgrade, a downgrade, a
card retried three days later and the proration of a mid-month switch, and each
is a place our own subtraction could drift from what the card is charged. A
host promised half price and billed full price is a refund and an apology; one
billed half of the *wrong* number is worse, because nobody notices.

Two halves, and the missing one would be invisible: applied to a live
subscription immediately, **and** carried into Checkout for a host who hasn't
subscribed yet. Without the second, somebody quoted a rate by a person meets
the list price on the payment page.

100% off is refused and points at Complimentary access. It is a grant wearing a
discount's clothes and the two behave differently everywhere that counts — a
comp needs no card, no subscription and no Stripe at all, whereas a "free"
subscription still needs a payment method that clears.

Shown to the host on /settings, in the status card and on every plan card
(list price struck through). All three cards, not just the one they're on: a
host deciding whether to upgrade needs to know what the upgrade costs *them*.

## 2026-08-13 — Refunds are a state, not an absence

`UpsellStatus` gained `refunded`. Deliberately its own state rather than a walk
back to `approved`: a refund happened, both parties saw it, and both should be
able to find it afterwards. Making it a state is also what stops it counting —
every filter asking for `"paid"` stops matching by itself, which is more
reliable than four places each remembering to exclude it. The compiler found
all three status maps needing the new word, which is what exhaustive
`Record<UpsellStatus, …>` types are for.

Learned from `charge.refunded` on the **connected accounts** destination, since
guest charges are created on the host's account. The charge carries no stay —
metadata lives on the Checkout session — so the payment intent is the thread
back, looked up on the connected account. Keyed to the session that was
actually refunded, because a repeatable extra can carry several paid lines.

`Mark refunded` sits beside the long-standing `Mark paid` for money handed back
outside Stripe. It records no amount: the host is saying it went back, not how
much, and inventing a figure would file a partial cash refund as a full one.

## 2026-08-14 — A stay keeps its own hours

`Reservation` gained optional `checkInTime`/`checkOutTime` ("HH:MM", blank =
property default), resolved through `stayCheckInTime`/`stayCheckOutTime` in
domain — the same override-with-fallback shape as the door code. Every clock
that mentions a stay reads through the resolvers: portal headline and times
card, guidebook tokens (`{{property.checkOutTime}}` keeps its name but answers
with the stay's truth), the code-release instant, the upsell hour-window, and
the cleaner's window from *both* sides — a sold late checkout starts the clean
later, the arriving guest's early check-in ends it sooner.

Editable in the booking drawer for synced stays too, on purpose: the channel
owns the dates, but "leave at noon instead" is agreed in a message thread the
iCal feed never reads. A time set equal to the property default is stored as
no override, so undo isn't a feature anyone has to learn.

## 2026-08-14 — The farewell flips at the checkout hour

`stayHasEnded()` (checkout instant, property timezone, override-aware) now
drives the portal's thank-you and headline: at 1:59 a paid 2pm late checkout
still reads "checkout is 2pm", at 2:01 "thank you for staying". Day-level
`hasDeparted` couldn't say this before midnight. The entry code deliberately
keeps day-level manners — a guest stepping back for a forgotten bag at five
past shouldn't meet a locked page.

## 2026-08-14 — When the window moves, the invite follows

Clean windows are derived, so changing a stay's times or dates moved them
silently — and an assigned cleaner kept a calendar entry for hours that no
longer existed. `cleaning-reinvite.ts` (lib, NOT a server action — it takes
accountId on trust) snapshots the windows a change can touch (the stay's
turnover + mid-stay, and a same-day predecessor's turnover), recomputes after
the write, and where one moved: ICS sequence bumped so the calendar updates in
place, same token so the old link still answers, status back to `invited`
because a yes to 11-to-3 is not a yes to 2-to-6. Self cleans move silently.

## 2026-08-14 — "Myself" is a valid answer on the schedule

`ReservationCleaning.status` gained `"self"`: no cleaner, no token, nothing
mailed, but covered. `jobState()` in cleaning.ts is now the single reader
(schedule rails, sidebar badge, lede arithmetic) — the page and the sidebar
had been answering "is anybody on it?" separately and drifted one status
apart the moment a fourth status arrived. Undo reuses withdrawal, with nobody
to notify. Turnovers also grew a Past tab (checkouts gone, newest first, no
controls — history takes no assignments) and every sample fence: rows,
dropdown, sidebar count.

## 2026-08-14 — Fine print is a field, not a paragraph

`Upsell.finePrint` — small faint text after the description on guest and host
cards, its own quiet input in both editors. A liability line pasted into the
description makes every card read like a rental agreement; a separate field
lets the sell stay warm and the caveat stay legible. Blank clears it.

## 2026-08-15 — Extras grew a full negotiation loop

A run of connected decisions, recorded together:

- **"withdrawn" is its own status** — the guest's change of mind, never
  the host's "declined"; spends nothing, counts nowhere, and leaves no
  trace on the guest page (the card simply returns to the shelf) while
  the host's queue keeps the ledger.
- **"Offer it again"** stamps a declined/refunded line `reopenedAt` —
  history stays, blocking stops. availableUpsells' everAsked and
  requestUpsell's dedup are two halves of one rule and must move
  together; the first time they didn't, the re-offered card's Request
  button silently no-opped behind the double-tap "ok".
- **Approval names its price** — prefilled at the ask, moves only down,
  zero waives and settles instantly; `discountedFrom` keeps the asked
  price beside the new one because a discount has to be visible to be
  felt (guest line, queue, drawer, email).
- **Typed asks** — an extra collects nothing, a message, a time, or a
  date at request; the picker composes a human string ("3:30pm") that
  travels as the ordinary note so every surface quotes it unchanged.
  Legacy asksNote reads as "text" through one upsellAsk() reader.
- **In-app Stripe refunds** — Refund… beside Mark refunded on paid rows,
  initiate-only (charge.refunded does the books, same as dashboard
  refunds); fee note: application fee is kept by default on refunds,
  flagged as a decision for the third-party-host era.
- **Manual arrival release** — the drawer can open door details past the
  timer and the ID hold (detailsReleasedAt); never invents a missing
  code, never reopens a finished stay. Built the week live Identity was
  pending Stripe's own "verify yourself" step.

## 2026-08-15 — The cleaner side became a small app

One standing link per cleaner (scheduleToken, minted once, SQL-guarded
against double-mint so the link can never rotate; edits preserve it).
The page: three hash tabs — Schedule (day-grouped agenda, Hospitable's
time-block-left card shape), Calendar (two month grids coloured by
answer: clay invited > sage accepted > grey declined; declined stays
visible so the answer can change until reassignment), Requests (inline
Accept/Decline, count on the chip, brass callout above). Answering an
invite lands on the schedule, not a spent confirmation page.

Saved-to-home-screen is first-class: a per-token webmanifest (start_url
is the token URL) makes Add to Home Screen a real standalone app, which
arms pull-to-refresh (standalone-only, animated pill) and the Badging
API count. Web Push closes the loop: subscriptions live on the Cleaner
(one per device, pruned on 404/410), pushes ride beside the emails on
assign/withdraw/window-move with the pending count as the badge, and
the auth scanner recognises getCleanerByScheduleToken as a caller
check. VAPID keys in env — regenerating them silently kills every
subscription, so they are permanent.

## 2026-08-15 — Samples fenced everywhere; the extras queue splits

The samples-are-furniture fence extended to the last two screens that
still mixed them in once a real property existed: Bookings (rows,
property dropdown, month calendars) and the Extras screen (queue,
stats, Sold breakdown). Same rule as Turnovers: a brand-new account
keeps the samples — an empty screen teaches nothing — and the first
real property retires them.

The extras queue itself split into Active | Past. First cut keyed the
split on the stay alone — a current guest's declined and withdrawn
lines stayed Active "because Offer it again still applies" — and
Cullan caught it immediately: those are exactly the clutter the split
was for. Final rule: a line's own no (declined, withdrawn, refunded)
files it under Past at once, and a departure or cancellation files
whatever is left; Active is only requests, unpaid approvals and paid
extras for guests here or on the way. Offer it again works from Past —
reopening is an act on history. Attention stats (needs approval,
approved-unpaid) read the active half only; Collected stays all-time.
The toggle is local component state, not the URL — the outer tabs own
the hash, and which half of a queue you're reading is a glance, not an
address.

## 2026-08-16 — Calendars grew up; the cleaning ledger became editable

The three calendars (bookings, turnovers, cleaner) converged on one
family: month grids to each property's own horizon (min two months,
capped at two years), 17px display-face month headings, rules between
stacked months on phones, past days struck grey — except an in-progress
stay, whose elapsed nights only fade so the block stays whole. Bookings
alternate blue/green by arrival order so back-to-back stays read apart
(the clash clay stays in code as a silent alarm, out of the legend);
block borders and corners sit only at a stay's true ends, so a week
wrap reads as one booking cut, not two. Hovering any night lights the
whole stay (component went client; today decided server-side so
hydration can't straddle midnight; names-only guest map crosses the
boundary). Past cleans keep one grey box, named in the tooltip and the
legend.

The cleaner got a Past tab (after Requests) — cleans file there on
their own after checkout, chips in past tense, same JobCard as the
schedule. The host's Past turnovers view keeps the picker on every row:
assigning a past clean *records* — server books it accepted on the
spot, no invite mail/push/ics for finished work — and can correct a
wrong name, not just fill a blank. The clean invite page grew the
guest's own arrival sections behind a tap, fill-ins and door code
resolved: the host chose the cleaner, and a cleaner who can't pass the
lobby cleans nothing. The reservation drawer's Extras figure counts
paid lines only.

## 2026-08-16 — Cleaning checklists

New entity (own jsonb table, `ckl_` ids): a checklist is name + sections
(title + steps) + propertyIds, authored on Turnovers' fourth tab with a
one-textarea-line-per-step editor, scoped by ticking properties (server
intersects posted ids with owned, seeds excluded). The cleaner's job
page shows each applicable list as a foldable card — sections fold to
steps, per-section and overall progress chips go green when complete —
and every tick persists to the job's cleaning record
(`checklistDone[checklistId] = ["section:step", …]`) under the row
lock, invite token as caller. Ticks live on the JOB, not the checklist:
two cleans of the same flat are two fresh lists, and no mail is sent —
a working list, not an audit.

## 2026-08-16 — Hospitable went live; the clean window hardened

Channel managers now dispatch through one seam (integrations/manager):
each adapter parses its own credential and speaks the shared Remote
vocabulary. Hospitable is real — PAT as the whole credential, v2 API,
per-property reservation queries filtered by checkout so in-residence
guests arrive too, everything read defensively (cents, ISO days,
status at reservation_status.current.category). PATs are fine while
Entrio's hosts are ourselves; the third-party era needs OAuth (TODO).
Migration rule: a booking already present via iCal (same confirmation
code, or same property+dates) is CONVERTED in place — portal link,
door code and extras survive — never duplicated, and the iCal planner
treats the converted row's nights as spoken for. The dead
ENTRIO_PROVIDER seam (integrations/index, mock) retired. Stubbed-API
test suite pins listing adoption, named-guest import and the twin
conversion.

Separately, the clean window is now capped at four hours from checkout
everywhere — a same-day arrival can only shorten it. And guests' phone
became optional at claim: email is the reachable channel, a demanded
number is how a made-up one gets recorded; a number offered is still
held to shape.

## 2026-08-17 — The welcome message sends itself, and bookings ring a doorbell

Auto-welcome: when the switch is on (Settings, under the connection),
a booking newly adopted from the channel manager gets the welcome
message posted into its own platform thread via Hospitable's message
API — name, arrival date and portal link filled by renderTemplate.
Sent on the ADOPT path only, so updates, cancellations, backfills,
in-residence stays and converted iCal twins can never trigger it;
receipted as welcomeSentAt; future arrivals only; toggle defaults off.
The template lives on the account; an untouched box stores null and
keeps following the default. Sending requires a PAT with message
permission — read-only tokens refuse with words naming the switch.

Webhooks land on /api/hospitable/webhook as a DOORBELL: the payload is
never read — any event triggers the same authenticated API pull — so
there are no signatures to verify and nothing to spoof; keyed by
ENTRIO_CRON_SECRET in the query string (their form sets no headers),
404 without it, 30s per-account cooldown. Verified live end-to-end
with Hospitable's test ring. Cron + open-the-app refresh remain the
safety net beneath it.

## 2026-08-17 — The pre-arrival nudge, the rehearsal, and the debt rule

Second message moment: pre-arrival, sent by the hourly cron once now is
within the host's configured lead (days+hours before the stay's own
check-in moment, property timezone, early check-ins included; default
24h, clamped 1h–14d, stored as preArrivalLeadMinutes). Receipted as
preArrivalSentAt; never after check-in; suppressed permanently when the
welcome landed within lead+6h of arrival (a last-minute booker isn't
messaged twice). Welcome sends that FAIL at adoption (read-only token,
rate limit) stamp welcomeDueAt — a debt the sweep repays while the stay
is ahead — so a bad token day can't mean a guest never hears from us,
and enabling the switch later still can't blast the back catalogue.

Testing without a write token: the Rehearsal panel (Settings, under the
message cards) renders every upcoming booking's exact message text and
send moment from the live templates and the same clock arithmetic —
read-only by construction. sendDueMessages(account, now) is the one
clock; the rehearsal imports its exported arithmetic rather than
duplicating it.
