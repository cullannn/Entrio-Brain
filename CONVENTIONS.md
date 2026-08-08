# Conventions

*Last updated: 2026-08-03*

## Design language

- Type: Cormorant Garamond (display) over Inter (sans). Palette: ink / bone /
  linen / parchment / brass / sage / clay, as Tailwind tokens.
- **Numerals need `.tnum`** — Cormorant defaults to old-style figures, which
  wobble in tables and codes.
- Interaction: **recessed = information, raised = tappable, filled = primary
  action.** Press states over hover — the portal is read on phones.
- The entry-code card is brass-wash with ink numerals, never dark.
- Affordances match physical reality: copy buttons only where a value can be
  pasted somewhere (smart-lock codes and wi-fi passwords yes; lockbox wheels
  and network names no).
- Chips/badges: one `Badge` component, `BadgeTone` exported so status maps
  elsewhere can't name a tone that doesn't exist. Labels describe **the
  subject, not the setting** ("Not done", never "Optional") and are kept ≤10
  chars — measured against the narrowest grid, where `min-width: 0` clips
  silently.
- Grid columns for table-like lists are **string literals** (Tailwind reads
  files as text; computed class strings generate no CSS), and header + row
  share the literal so they can't drift.
- The guest portal is a four-tab app; add a screen or a card, never extend a
  scroll. Tab state lives in the URL hash so Back works and tabs deep-link.

## Words in the product

- **Name what the reader sees, not what the system does.** "Arrival details
  unlock" was our word for a scheduled reveal and it reached seventeen
  places — the editor, the guidebook, the token panel, the checklist.
  Nothing on a lockbox unlocks; the details *appear* at a time the host
  picked. Internal metaphors leak into UI copy one label at a time and are
  invisible to whoever coined them.
- Layout vocabulary is ours, not theirs: "hero", "chip", "token", "field
  group". A host has a main photo, a button, their own details, a section.
- A feature named for its mechanism gets renamed for its payoff the first
  time somebody asks what it means. "Fill-ins" → "Their own details",
  which also says why you'd use it: write once, every guest reads their own.
- Copy that names a sample, a plan or a screen should read the name from
  the same source the screen does. A checklist that pointed at "The Luxury"
  survived two renames of the property it meant.
- Instructions lead with what the host does. "Step-by-step directions from
  the street to inside" beats "the Arrival tab contains…".

## Dates, time, timezones

- **Date-only strings go through `parseDay()`, never `new Date(iso)`** — the
  UTC parse renders yesterday west of Greenwich.
- The app runs on **local dates by design**; tests must not use
  `toISOString()` for "today" or they pass in the morning and fail at night.
- Instants (release, close, sale windows) are computed with `zonedTime(day,
  hhmm, propertyTimezone)`. Hour-granular logic must never be derived from
  day arithmetic (see INCIDENTS).
- On turnover days, "arriving/departing today" beats "night N of M"; a
  cancelled stay beats everything.

## Data rules

- **Nothing real in the seed, ever.** Fictional properties only; the host's
  real details live in gitignored `.data/` + `public/photos/private/`,
  applied locally by `npm run seed:mine`.
- The seed regenerates relative to today each boot; user changes live in the
  overlay. Consequence: **never hand-edit `.data/` while the dev server
  runs** — it holds the overlay in memory and writes it back (`pkill` first).
- Store objects are never mutated in place (SQLite write-diffing depends on
  reference identity).
- Channel imports append/update by externalId only; host work is never
  overwritten; cancelled rows are kept, not deleted.

## Testing

- No framework. `npm test` → `tests/run.mjs` spawns each `tests/*.test.ts`
  under `node --experimental-strip-types`; a test prints `✓/✗` and exits
  non-zero. Keeping the next test near-free to write is the point.
- **The runner sets `ENTRIO_DATA_FILE`; tests must never set it themselves**
  (ESM hoisting means the store resolves its path before the assignment
  runs — the test passes while writing into real data).
- `tests/alias-hook.mjs` bridges `@/` aliases, extensionless imports, and
  stubs `server-only` / `next/cache` so real modules are testable.
- Wire-level guarantees get wire-level audits (`audit:guest`,
  `audit:bundle`), and an audit must **fail loudly when it cannot run** —
  a green result must state what it checked.
- Invariants that code review keeps missing become source-scanning tests
  with documented allowlists (`id-check-callers` is the template).
- UI claims get verified in a real browser at real widths; 414 px via
  same-origin iframe (`resize_window` is unreliable; zoom changes layout but
  not media queries).

## Code & process

- Commit messages are prose-first: what broke, why it was wrong, why this
  shape. The brain (this repo) carries the cross-commit narrative.
- Comments explain *why*, often naming the bug that taught the lesson.
- Server-action failures the user should see are **returned results**
  (`{ok:false, error}`), not thrown Errors — Next strips thrown messages in
  production.
- Prefer one domain function over repeated inline logic the moment two call
  sites disagree (labels, gaps lists, gates all got this treatment).
- `npm run lint`, `npx tsc --noEmit`, `npm test` green before commit; the
  audits run against a dev server when the change touches the wire.
