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
