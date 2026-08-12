# Incident log

Bugs that actually happened, with root causes and the lesson each one left
behind. The fixes are in git; the lessons are why this file exists. Newest
last — append.

---

### 2026-07-31 — AVIF that decodes but paints transparent
macOS `sips` can emit an AVIF with valid headers, correct dimensions, and
invisible pixels; `<picture>` selects by MIME and never falls back.
**Lesson:** verify images by drawing to a canvas and sampling pixels —
`img.complete` and `naturalWidth` both lie. Encoded into `npm run photos`
(pixel-verified manifest).

### 2026-08-02 — "31 guest names" that were two strings
A Hostaway feed was checked for PII by masking letters; 31 identical
"by Hostaway" summaries were read as 31 real names, and vendor boilerplate
nearly shipped as guests' display names.
**Lesson:** when masking data to inspect it, count *distinct* values first.

### 2026-08-02 — Cancelled stay on the Now tab, "night 11 of 17", in green
Status was written once at creation and never advanced; timing was derived
from dates, which a cancelled booking still has.
**Lesson:** dates answer "where is this stay"; the stored status is
authoritative only for what dates can't express (cancelled, inquiry) — and
cancelled must win over every other rule.

### 2026-08-02 — `* { min-width: 0 }` outside a cascade layer
Killed every `min-w-*` utility in the app: unlayered rules beat
`@layer utilities` regardless of specificity in Tailwind v4.
**Lesson:** resets go in `@layer base`, always.

### 2026-08-02 — "Clear the sample" would have deleted a real property
A seeded property that a host has renamed and wired to a real calendar is
still a seed row underneath; the button dropped all seed rows.
**Lesson:** the offer to clear sample data must withdraw itself the moment a
seed row carries real work (a booking from the host's own channel).

### 2026-08-03 — The guidebook vanished (data intact on disk)
`seed:mine` rebuilt the guidebook from ids scraped out of the seed *source*
(`[{ id }]`), betting the store would fill in the rest. It didn't; every
section lost its `chapter`; both the editor and portal find sections by
filtering on chapter, so all six were on disk and on no screen.
**Lessons:** (1) import fixtures, never parse them as text; (2) merge
weakest-first (seed → stored patch → private file) so damage self-heals;
(3) data that filters can orphan needs a read-side safety net — chapterless
sections now land in "house" instead of nowhere.

### 2026-08-03 — Entitlement bug, three times, same shape
`!verification.idVerified` read raw at three call sites (door gate, guest
checklist, host dashboard), ignoring plan entitlement and property mode.
Symptoms ranged from a guest stranded at the door to a Basic host nagged
weekly about a feature they don't have.
**Lesson:** when a check depends on facts from two places (plan + property),
give it one domain function and *forbid* the raw read — enforced by a
source-scanning test with a documented allowlist.

### 2026-08-03 — Hydration error on the only door into the app
A `<form>` inside a `<p>` on `/verify`: the HTML parser closes the paragraph
first, server markup ≠ DOM, React throws on every visit — on the page whose
client-side auto-submit sends the OTP.
**Lesson:** invalid HTML nesting isn't cosmetic in React; the parser
rewrites it and hydration pays.

### 2026-08-03 — "Add a property" made two blank properties
Create, refresh, then scroll-to-card after 120 ms — a bet on a server round
trip. Losing it left no visible change, so the natural move was clicking
again.
**Lesson:** navigate to the thing you created; never sequence UI on a timer
against a network.

### 2026-08-03 — The audit that certified nothing
`audit:guest` (the wire-level check on the core promise) had two independent
false-pass modes: it never checked HTTP status (404 pages "withheld"
everything), and its seed parser searched *backwards* 1200 chars, matching
the previous reservation's door code — so it grepped pages for a number that
appears nowhere while a real code sat on the wire.
**Lessons:** (1) a guard must prove the page is a page before concluding
anything from its absence of secrets; (2) "all clear" must state what it
cleared; (3) parse fixtures forward within the owning object, or better,
import them.

### 2026-08-03 — Live entry-code leak on the default plan
Three deliberate behaviours composed: release-on-booking (sample property) +
`idCheck` defaulting to mandatory + identity checks being Plus-only, which
collapses the gate on Basic. Both gates open → code, wi-fi and unit served
to an unverified guest six days early, on every Basic account. Hidden by the
broken audit above.
**Lessons:** (1) safe-in-isolation decisions need a composition check — now
`nothingWithholdsCode()` warns on the property page; (2) release-on-booking
must never be a shipped default.

### 2026-08-03 — Top-level merge destroyed the identity record
`patchReservation(id, { verification: { idVerified } })` replaced the whole
object, losing the Stripe session id and failed/mismatch flags — downgrading
"Failed, message them now" to "Awaiting", permanently orphaning the guest's
retry.
**Lesson:** patch semantics are top-level; spread nested objects explicitly.
Same family: writing a pre-`await` snapshot back after a network call reverts
anything a webhook wrote in between — fixed later the same day via
`mutateReservation` (see the concurrency entry below).

### 2026-08-03 — Hour-granular sale windows quantised to days
Upsell availability compared midnight-to-midnight, so an offset of "-4
hours" could never exist; Early arrival ($75) vanished at midnight before
arrival day — the exact morning it sells.
**Lesson:** measure "hours until" against the real instant (zoned check-in
time), never day arithmetic; DST skew disappears with it.

### 2026-08-03 — The async store let concurrent writes clobber each other
A reservation's extras and verification record are lists inside its own JSON,
so changing one item is a read-modify-write. Synchronous, that was one tick;
on Postgres every step is a network hop, so two requests read the same base and
the second write erased the first — two guests requesting different extras, or
a Stripe webhook landing during a host override, and one silently vanished. The
same shape was the deferred "stale-snapshot across await": identity writes
spread the verification read *before* a Stripe round trip.
**Lesson:** read-modify-write of a row's own nested list must hold a row lock.
`mutateReservation` wraps `SELECT … FOR UPDATE` + merge in one transaction; every
upsell/verification write moved onto it. Cross-row invariants (inventory across
different reservations) a row lock can't cover — left open on purpose, because a
request is an ask and only one is ever approved. Test: `concurrency.test.ts`
reproduces the clobber and proves the lock fixes it.

### 2026-08-03 — Assorted, one line each
`writeFileSync` truncate-in-place + silent corrupt-file recovery = full data
loss path (atomic rename + loud failure now; then SQLite). · Guidebook
template tokens (`{{reservation.doorCode}}`) bypassed the gate the entry
card enforced. · `NaN` payout became $0 silently. · A guest's upsell request
had no error path — sold-out threw into a transition and froze the button at
"Asking…". · Preview preferred real bookings, which silently removed the
before/after toggle. · Verification email promised "no account will be
created", which was false.

### 2026-08-03 — R2 uploads 411'd inside Next, worked everywhere else
The R2 PutObject returned `411 MissingContentLength` from the app, while the
exact same code — Uint8Array body, Blob body, even a manual Content-Length
header — returned 200 from a standalone node script. R2 rejects a PutObject
with no Content-Length, and **Next.js patches global `fetch`** so a byte-buffer
body goes out chunked with no length; none of the body forms fixed it because
the mangling is in the transport, not the body.
**Lesson:** when a network call behaves differently in the app than in a probe,
suspect the runtime's patched globals before the payload. The fix keeps
`aws4fetch` for the SigV4 signature but sends the PUT over `node:https`, which
sets Content-Length and never touches Next's fetch. Also why the probe lied:
reproduce inside the real runtime, not a clean one.

### 2026-08-03 — Clearing a jsonb field with `undefined` clears nothing (×5)
A test-coverage pass caught it: `patchAccount(id, { resetTokenHash: undefined })`
is a silent no-op. `JSON.stringify` drops undefined keys, so the `data || $x`
merge receives `{}` and the old value survives. Five clear-sites relied on it and
quietly kept stale state — worst: a spent password-reset link stayed live for its
full hour (a forwarded email = account takeover). Also: a used email-verify code
lingered, a landed plan downgrade advertised "switching to Basic on …" forever,
and **"unlink listing" never cleared `externalRef`, so the next channel sync
re-matched the property**. The coverage pass found three; a grep for the class
found two more.
**Lessons:** (1) to clear a jsonb field you must write `null` (which the merge
stores and every reader treats as absent), never `undefined` — now documented on
`patchAccount`/`patchProperty`, whose types accept null so a clear reads as one;
(2) a bug found at one site of a mechanical class is a prompt to sweep for the
rest, not fix the one; (3) writing a test that *documents* a bug (green, asserting
current behaviour) is how a found-but-unfixed bug survives to be fixed on purpose.

### 2026-08-03 — Checkout redirected to 0.0.0.0:PORT on the first live deploy
The live smoke-test's very first subscription paid fine (the webhook recorded
Plus), but Stripe sent the browser back to `https://0.0.0.0:10000/…` —
unreachable. The checkout `success_url` was built from `currentOrigin()`, which
read the **Host header**, and behind Render's proxy that header is the
container's own bind address (`0.0.0.0:PORT`). The app had *already* decided
"build URLs from config, never the Host header" for reset links — but that rule
never reached the billing origin, and worse, the same `currentOrigin()` was
**copy-pasted** into the guest upsell checkout, so a guest paying would have
bounced the same way.
**Lessons:** (1) a security/robustness rule adopted in one place (reset links)
has to be applied everywhere it's relevant — a URL built from the Host header is
the same mistake whether it's a reset link or a redirect; (2) copy-pasted
helpers spread the bug — both origins now share one `requestOrigin()` (prefers
`ENTRIO_SITE_URL`); (3) the first live deploy is where proxy/host assumptions
finally get tested — smoke-test the real environment, not just the code.

**And it took a second pass.** Fixing the two `currentOrigin()` helpers wasn't
enough: the two routes Stripe returns *to* — the guest `/stay/…/paid` and the
host `/billing/confirm` — each did a *second* redirect with
`new URL(path, request.nextUrl.origin)`, which behind the proxy is `0.0.0.0`
again. So the return_url was finally right but the guest was bounced anyway. The
tell was the final URL being the portal (`#stay`), not `/paid`. **Extra lesson:**
`request.nextUrl.origin`/`request.url` are the *same* untrusted-origin trap as
the Host header, and a partial sweep (`currentOrigin`, `get("host")`) missed
them. A scan test (`redirect-origin.test.ts`) now forbids building any redirect
base from the request origin — this class had three instances before it was
caged.

### 2026-08-03 — The endpoint that hides on purpose, diagnosed as broken
The calendar-sync endpoint answers **404 to any request without the right
bearer** — deliberately, and identically for "secret not configured" and
"wrong token", so it never advertises that it exists. Standing up the new
cron, an unauthenticated probe curl got 404 and that was read as
"misconfigured", sending the operator off to re-verify env vars that were
already correct (with a wrong method-mismatch theory offered along the way —
the route handles both GET and POST). The first *authorized* run returned 200;
nothing had ever been broken.
**Lessons:** (1) an endpoint designed to hide cannot be probed from outside —
404 *is* its healthy state; the only meaningful check is through the
authorized path (here, the cron's own trigger). (2) Read the route's failure
branches before diagnosing from a probe's status code; the misdiagnosis came
from remembering only one of the two ways it 404s. (3) If self-hiding ever
hurts operations, the fix is a separate authenticated health probe — not a
chattier 404.

### 2026-08-04 — A guest without an email couldn't pay
First smoke test on the real domain: Pay → "Couldn't open the payment page."
The checkout passed the guest's email straight through to Stripe. The SDK
silently drops an `undefined` param, but a guest row saved with a *blank*
email string sends `customer_email: ""` on the wire, which Stripe rejects
before the session exists — and Next redacts thrown server-action messages in
production, so the guest saw only the generic retry line for a state retrying
can't fix. A host-created booking with the email field left empty is a
completely normal booking.
**Lessons:** (1) an "optional" field has two absent shapes — missing and
empty — and an SDK that quietly handles one will happily ship the other;
normalize at the provider boundary (`?.trim() || undefined`), where the
external API's rule actually lives. (2) In production the real error only
exists server-side (logs / the alert email); the client's friendly fallback
is by design, so diagnosis has to start at the logs, not the message. (3)
Checkout collects an email itself when none is supplied, so omission was
always the correct behaviour, not a degraded one.

### 2026-08-04 — The homepage deck ignored thumbs and outdrew the header
Three mobile-only faults shipped in the homepage showcase, all invisible on
a desktop. (1) The page panned sideways: the deck's fanned side screens are
translated far past the column on purpose, and past the viewport edge that
bleed becomes a horizontal scroll area. Root fix: `overflow-x: clip` on the
page root — clip, not hidden, because hidden makes a scroll container and
quietly changes sticky/scroll behaviour, while clip only forbids the axis.
(2) Swiping the deck did nothing on a real phone even though the stage
declared `touch-action: pan-y`: the browser resolves touch-action by walking
up from the touched element only as far as the *nearest scroll container*,
and the mock's own scrollable screen is one — so a finger on the screen
never saw the stage's declaration, the browser claimed the gesture, and the
drag was pointercancelled. Any scrollable child inside a custom-gesture
surface must carry the touch-action itself. (3) The mocks drew over the
sticky site header: the fan layers itself with z-indexes in the hundreds,
and without `isolation: isolate` on the deck root those values compete in
the page's stacking context and beat the header's modest z-50. Decorative
z-index ladders stay private behind an isolated ancestor.
**Lesson:** every one of these passes with a mouse on a wide window;
touch-action, viewport-edge bleed, and stacking-context leaks only show up
on the phone itself. The 414px check has to include *touching*, not just
looking.

## 2026-08-05 — The tab bar that could not reach the glass

The host app's bottom tab bar floated above the bottom of the screen on a
home-screen (standalone) install, with a dead band below it, and survived
four "fixes" aimed at CSS. Ground truth, obtained by shipping a one-deploy
diagnostic readout into the bar itself: the OS sized the standalone window
as screen-height *minus the status-bar inset* but anchored it at the top of
the glass — so a band at the bottom equal to the *top* inset sits outside
the webview and no CSS can ever paint or occupy it. Meanwhile
`env(safe-area-inset-bottom)` still reported the full hardware inset even
though the home indicator lay outside the window, so honest-looking
safe-area padding double-spaced the bar.

Along the way, three transferable fixes:
- Never trigger work from inside a setState updater. The pull-to-refresh
  release called startTransition inside a `setPull` updater; React may
  replay or drop updater work, which showed as a refresh veil over a
  refresh that never began. Decide in the event handler, read gesture
  state from a ref.
- Any full-screen "busy" overlay needs a bail-out timer. A wedged RSC
  refetch (dead spot, server mid-deploy) left the page blurred forever;
  the veil now dismisses itself after eight seconds regardless.
- Safe-area clearance is arithmetic, not faith: needed padding =
  env inset − (screen.height − window.innerHeight), re-measured on resize
  because a standalone launch settles its window size after first render.

**Lesson:** when a layout bug survives two correct-looking CSS fixes, stop
inferring from screenshots and make the device print its numbers —
viewport heights, env insets, the vh units, standalone flags. One readout
ended a five-round guessing loop. And when the platform genuinely
withholds screen from the page, change the design to own the gap (the tab
bar became a floating pill over a continuous field) instead of faking
reach that isn't there. Pull-to-refresh refetches data, never new client
code — a "still broken" report after a deploy may just be the old bundle;
have the phone fully relaunch the app before judging.

## 2026-08-06 — Five fixes into a corpse

The live preview kept "losing its top" after autosaves, and five
successive scroll fixes (anchor stripping, scrollTo resets, restoration
disarming) each failed to take. The villain was environmental: a corrupted
Turbopack dev cache had broken the stay page's chunk ("CJS module can't be
async"), so the iframe rendered without working JavaScript — every fix
shipped into a page that couldn't run it. Production builds were clean the
entire time. Clearing .next and restarting the dev server revived
hydration; two genuine bugs remained (a scrollIntoView block:"start"
ramming revealed rows to the top, and scroll anchoring drifting ~one bar
height post-hydration) and fell in one pass once measurements were real.
**Lesson:** when consecutive fixes don't take, stop shipping and read the
browser console — a dead-hydration page swallows fixes indefinitely, and
"my change did nothing" is itself diagnostic data pointing at the
environment. Verify with live measurements (scrollY, element rects), not
another screenshot round-trip. And check whether prod even shares the
symptom before treating a dev-only ghost as a product bug.

## 2026-08-07 — The picker that threw the photograph away

Section photo uploads did nothing from the file picker, while dragging the
identical JPG into the same box worked. Cause: `input.onChange` read
`e.target.files` into a variable, then set `e.target.value = ""` (the usual
trick so re-picking the same file still fires a change) — but `files` is a
**live FileList owned by the input**, and clearing the value empties it.
The array we then iterated was empty. Drag-and-drop reads
`e.dataTransfer.files`, its own list, which is exactly why it kept working
and hid the bug. Fix: `Array.from(e.target.files ?? [])` BEFORE the reset.
A sibling uploader that grabbed `files?.[0]` into a File first was
unaffected — a File reference survives the reset; the list does not.
**Lesson:** treat FileList (and any live DOM collection) as a view, not a
snapshot, and copy before mutating its owner. When two paths into the same
function disagree, the difference is in what each path *passes*, not in the
function.

## 2026-08-07 — The marketing claim the product couldn't keep

A persona review of the homepage caught it, not a test: the page said a
guest's entry code "stops working after checkout" and that "last month's
guest can't get back in" — two sentences below the promise that no smart
lock is needed. Both were false, and the second sentence is what made the
first one false. The product hides the code from the guest's page at the
end of checkout day; with a mechanical lockbox the combination itself
keeps opening the door until a human changes it. The copy had quietly
promoted a display rule into a security guarantee.
**Lesson:** any sentence about what a guest *can't* do is a claim about
the physical world, and has to be traced to the code that enforces it
before it ships. Marketing copy deserves the same "where is this
enforced?" question as an authorisation check — and the safe rewrite is
usually the literal behaviour ("the code is shown only during their
stay"), which sells nearly as well and is true.

## 2026-08-11 — The save that vanished, and the black page that hid why

Creating a property on production intermittently gave a black unbranded
error page; refreshing showed nothing had saved; the identical click
worked on the second try. Reported as "happens multiple times".

Two independent defects, and only together do they explain the shape:

1. **The `pg` Pool had no `error` listener.** node-postgres emits `error`
   on the pool when a *checked-in, idle* client fails — managed Postgres
   restarting, a proxy dropping a socket, the network blinking between
   requests. Unhandled, that is an uncaught exception, and Node exits.
   The in-flight request dies with the process (write lost), and the next
   request boots a fresh process (works). That alternation is the
   signature: **a bug that "fixes itself on retry" is a process dying, not
   a race.**
2. **No `error.tsx` anywhere in the app**, so Next served its own default
   error UI. The user could not tell a crash from a validation failure,
   and — worse — was never told the save may not have landed.

Fixes: a pool `error` listener that logs and survives, `keepAlive: true`,
one retry in `query()`/`transaction()` gated on connection-level failures
only, and branded error + global-error boundaries that say plainly "what
you just saved may not have saved" and carry the digest for log lookup.

**Lessons:**
- Any long-lived connection pool needs an `error` handler *at creation*.
  The default behaviour is to crash the process, and it will only ever
  fire in production, where idle connections actually get dropped.
- Retry only what is safe to retry, and write down why it is safe. Here:
  app-generated ids, `ON CONFLICT DO UPDATE`, jsonb merges — no counters.
  The reasoning belongs next to the retry so the next writer of a
  non-idempotent statement sees it.
- Ship an error boundary before shipping to users. An unstyled black page
  is not just off-brand; it withholds the one fact the user needs.

## 2026-08-11 — The dashboard counted our properties as theirs

**Symptom.** The host home page said "6 properties" and drew six rows in the
three-week strip for a host who owned none of them — all six were the shipped
samples. Occupancy was computed over the same six, so a real property with a
full month read as a sixth of the truth.

**Cause.** `HostChrome` and `setupSteps` had already learned to filter the
samples out (`own = properties.filter(p => !isSeedPropertyId(p.id))`); the
dashboard hadn't. It predates samples becoming permanent furniture, when
`properties.length === 0` was a meaningful test.

**Lesson.** When a set stops being temporary, every count over it becomes a
claim about the host's business. The filter is the convention; the fix was to
apply it at the last place still counting ours as theirs — and to give the
strip an empty state, because a ruler of dates over blank space reads as a
broken component rather than an empty one.

## 2026-08-11 — A connection to a mode we can no longer see

**Symptom.** An account emptied through the new admin portal still showed a
connected Stripe account — one created against the sandbox key.

**Two causes, and the second is the dangerous one.**

1. `resetAccountData` deliberately leaves Stripe alone. That default is right —
   quietly detaching somebody's live subscription because they asked for a
   clean dashboard is a much bigger thing than they asked for — but a sandbox
   connection has to go before the live switch, so it's now an opt-in checkbox
   and two standalone buttons.

2. **`patchAccount` cannot clear a Stripe id at all.** It writes the indexed
   columns as `COALESCE($2::jsonb->>'stripeAccountId', stripe_account_id)`, so
   passing null clears the jsonb key and silently leaves the *column* pointing
   at the old id. `getAccountByStripeAccount` reads the column — a webhook
   about the abandoned account would still have resolved to that host.
   Clearing now has its own function that nulls key and column together.

**The lesson that generalises.** Stripe's two modes are two separate sets of
books: an `acct_`/`cus_`/`sub_` made under a test key does not resolve under a
live one. Nothing in the app noticed, because the field is a non-empty string
either way — the host is told they're connected and taking payments, and the
first thing to say otherwise is a guest's declined extra. At a live cutover
this is not an edge case; it is *every* account that ever connected or
subscribed, all at once. The portal now verifies each id against the key
actually configured and lists the stranded accounts on /admin/stripe.

**Generalises further:** a mirrored column beside a jsonb blob needs its own
clearing path. COALESCE-on-update makes "set" and "unset" asymmetric, and the
asymmetry is invisible until something reads the column.
