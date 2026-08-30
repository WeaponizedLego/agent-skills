# CLAUDE.md

<!-- SETUP — delete this comment block once done:
  1. You copied the contents of claude-boilerplate/ into this project's root.
  2. Fill in every [ ... ] slot below (Stack, conventions, tooling bans).
  3. Project already had a CLAUDE.md? Merge: keep your existing content, bring
     over the sections below (Skills, Feature workflow, Precedence, Shortcut
     convention) and reconcile the rest by hand.
  4. Keep `claude-artifacts/` in .gitignore (the copied .gitignore has it;
     feature-chain re-adds it if it ever goes missing). -->

## Stack

<!-- Be specific about versions — this is the most-used section in the file. -->

- Language: [ e.g. TypeScript (strict) / Go 1.26 / Python 3.12 ]
- Framework / runtime: [ e.g. Nuxt 4 / Gin / FastAPI / plain Node ]
- Package manager: [ e.g. pnpm / go mod / uv ]
- Deploy target: [ e.g. Node 26 / a single binary / Lambda ]
- Testing: [ e.g. Vitest / go test / pytest ]
- Libs we commit to: [ e.g. Zod, Pinia / sqlc / Pydantic ]

## How to work here

**Ponytail is the default lens** (level: [ full ] — `lite` / `full` / `ultra`):
the laziest solution that actually works. Question whether the task needs to
exist (YAGNI), prefer the shortest correct diff — but laziness runs *with* the
stack's grain: **stdlib → framework idiom → dependency we already have → only
then new code**. The blessed way IS the lazy way here; never hand-roll a vanilla
version of something the framework ships to save a dependency we already pay for.

## Skills in this project

Loaded from `.claude/skills/`. They route on trigger phrases, but use them
**proactively** — the moments below call for a skill even when the user doesn't
name one:

| Skill | Reach for it when |
|---|---|
| **feature-chain** | The user describes a non-trivial feature to build — propose the chain before writing code. Never for bugfixes or one-liners. |
| **feature-docs** | The user says a feature is done ("wrap it up", "this is done"). Judges if docs are warranted at all — "nothing worth keeping" is a valid outcome. |
| **docs-as-static-html** | Any standalone doc, runbook, or handoff page — and the rendering step of feature-docs. Output always lands in `docs/`. |
| **ponytail** | Active by default (see above). Satellites: **ponytail-review** (diff over-engineering check), **ponytail-audit** (whole-repo bloat scan), **ponytail-debt** (ledger of deferred `ponytail:` shortcuts), **ponytail-gain** / **ponytail-help** (scoreboard / reference). |

## Feature workflow

Non-trivial features go through the artifact chain before code:

```
claude-artifacts/<slug>/01-intent.md → 02-spec.md → 03-plan.md → code → feature-docs
```

- **feature-chain** drives it; the spec is the single user sign-off gate.
- Everything under `claude-artifacts/` is gitignored scratch — **never commit
  it**. It forces thinking before code and feeds the docs pass; the durable
  layer is `docs/`, written only by feature-docs, only when the feature
  warrants it.
- In `docs/`, markdown is the source of truth; `.html` files are renders —
  regenerate them, never hand-edit them.

## Precedence when guidance conflicts

Apply the first rule that settles it:

1. **An explicit instruction in the current request.** Always wins.
2. **The non-negotiables** below. Never simplified away.
3. **This project's conventions** (next section). Don't relitigate per-task.
4. **The stack's own idiom** (stdlib → framework → existing dep).
5. **Ponytail's ladder** for everything the rules above don't decide.

## Project conventions

<!-- YOUR stack's idioms — these override raw line-minimization. Keep short. -->

- [ Use the framework's data layer before adding an HTTP/DB client. ]
- [ Follow the framework's project layout; don't invent structure. ]
- [ Validation through one shared schema/type source. ]
- [ ... ]

## Non-negotiables

Never simplified away, regardless of ponytail level:

- Input validation at trust boundaries.
- Error handling that prevents data loss or corrupt state.
- Security: authz checks, secret handling, injection-safe queries.
- Accessibility basics (where there's a UI).
- Anything the user explicitly asked for in full — build it, no re-arguing.
- Tooling bans: [ e.g. for JS/TS: pnpm only, NEVER npm. Add hard rules here. ]

## The shortcut convention

A deliberate simplification with a known ceiling is marked inline so it gets
tracked instead of rotting silently:

```
// ponytail: <the limit>, <what triggers the upgrade>
// ponytail: in-memory cache, swap for Redis if we run >1 instance
```

Name the trigger — a shortcut with no upgrade path quietly becomes permanent.
These markers are read downstream: **feature-docs** harvests them into each
doc's Known Limitations section, and **ponytail-debt** builds its ledger from
them.

## What still earns a test

Non-trivial logic (a branch, a parser, a money/auth path) leaves one runnable
check behind — the smallest thing that fails if the logic breaks, using the
project's test setup ([ fill in ]). No elaborate fixtures unless asked; trivial
one-liners need no test.
