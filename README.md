# Entrio Brain

The long-term memory of [Entrio](https://github.com/cullannn/Entrio) — a
guest-experience platform for short-let hosts (guest check-in links, arrival
guidebook, and approve-then-pay extras). Code lives in the main repo; **why the
code is the way it is** lives here.

## Index

| File | What it holds |
|---|---|
| [ARCHITECTURE.md](ARCHITECTURE.md) | System shape, data model, the store contract, key flows |
| [DECISIONS.md](DECISIONS.md) | Dated log of every decision that shaped the product |
| [SECURITY.md](SECURITY.md) | The gate model, tenancy rules, and how they're enforced |
| [INCIDENTS.md](INCIDENTS.md) | Bugs that happened, root causes, and the lessons encoded |
| [SCALING.md](SCALING.md) | The 1000-host assessment and the staged upgrade plan |
| [CONVENTIONS.md](CONVENTIONS.md) | Design language, testing rules, code conventions |
| [INTEGRATIONS.md](INTEGRATIONS.md) | Stripe, Google Places, Anthropic, iCal, Resend, channel managers |
| [TODO.md](TODO.md) | Open items, deferred findings, deploy checklist |

## Maintenance protocol

This repo is only useful if it stays current. The working rule, also written
into the main repo's `AGENTS.md`:

- **After any session that changes architecture, scope, or a key behaviour** —
  append to `DECISIONS.md` (dated) and update `ARCHITECTURE.md` if the shape
  moved.
- **After any real bug** — add it to `INCIDENTS.md`: what happened, root cause,
  the lesson. The lesson is the point; the fix is in git.
- **When work is deferred** — put it in `TODO.md` with enough context that a
  future session can pick it up cold.
- Commit messages here can be one line. The prose lives in the files.

## The one hard rule

**Nothing real goes in this repo.** No real addresses, door codes, lockbox
locations, wi-fi passwords, guest names, or live tokens. The main repo keeps
the host's real property in gitignored local files (`.data/`,
`public/photos/private/`) for exactly this reason, and this repo is public.
Describe the pattern, never the values.
