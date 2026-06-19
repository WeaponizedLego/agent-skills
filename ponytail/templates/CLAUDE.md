# CLAUDE.md

> Baseline template. Fill in the `[ ... ]` slots and delete what you don't need.
> The goal: keep ponytail's bias toward less code, but let _this project's_
> stack define what "less" looks like. Nothing below assumes a particular
> language or framework — the worked examples at the end are swappable.

## Stack

<!-- The single most useful thing in this file. Be specific about versions. -->

- Language: [ e.g. TypeScript (strict) / Go 1.26 / Python 3.12 ]
- Framework / runtime: [ e.g. Nuxt 4 / Gin / FastAPI / plain Node ]
- Package manager: [ e.g. pnpm / go mod / uv ] — **for JS/TS: pnpm only. NEVER use `npm` (no `npm install`, no `npm run`, no `package-lock.json`).**
- Deploy target: [ e.g. Node 26 / a single binary / Lambda ]
- Testing: [ e.g. Vitest / go test / pytest ]
- Other libs we commit to: [ e.g. Zod, Pinia / sqlc / Pydantic ]

## How to work here

**Ponytail is the default lens** (level: [ full ] — `lite` / `full` / `ultra`).
The laziest solution that actually works: question whether the task needs to
exist (YAGNI), reach for what we already have before writing new code, prefer
the shortest diff that's correct.

**But laziness runs _with_ the grain, not against it.** "The laziest correct
solution" means the one our stack already ships — read in this order:

1. The **language standard library / built-ins**.
2. The **framework or runtime's idiomatic way** (its blessed API, not a
   hand-rolled vanilla version, and not a new dependency that duplicates it).
3. A **dependency we already depend on**.

Only past all three do you write new code. This is just ponytail's ladder
expressed in our stack's terms — it never pushes us off the happy path to save
a few lines.

## Precedence when guidance conflicts

Apply the first rule that settles it:

1. **An explicit instruction in the current request.** Always wins.
2. **The non-negotiables** (see below): security, input validation at trust
   boundaries, error handling that prevents data loss, accessibility. Never
   simplified away.
3. **This project's conventions** (the section below). Deliberate choices;
   don't relitigate them per-task.
4. **The stack's own best practice / idiom** (stdlib → framework → existing dep,
   as above). If there's a blessed way, that _is_ the lazy path here.
5. **Ponytail's ladder** for everything the rules above don't decide.

## Project conventions

<!-- Fill in with YOUR stack's idioms. These override raw line-minimization.
     Keep it short — every line here is loaded every session.
     The bracketed items are illustrations across stacks; replace with yours. -->

- [ Use the framework's data layer (e.g. `useFetch`, `database/sql`, an ORM we
  already chose) before adding an HTTP/DB client. ]
- [ Follow the framework's project layout and conventions rather than inventing
  our own structure. ]
- [ Validation through one shared schema/type source. ]
- [ State / config handled the framework's blessed way; no parallel hand-rolled
  globals. ]
- [ ... ]

## When NOT to be lazy (non-negotiable)

Never simplify these away, regardless of level:

- Input validation at trust boundaries.
- Error handling that prevents data loss or corrupt state.
- Security: authz checks, secret handling, injection-safe queries.
- Accessibility basics (where there's a UI).
- Anything the user explicitly asked for in full.
- **Tooling bans:** [ for JS/TS, `pnpm` only — NEVER `npm`. Add other hard
  tooling rules here. ] Ponytail's "use what's installed" never overrides this.

If the user insists on the verbose/full version, build it — no re-arguing.

## The shortcut convention

When a deliberate simplification has a known ceiling, mark it inline so it can
be tracked and revisited instead of rotting silently:

```
// ponytail: <the limit>, <what triggers the upgrade>     (// for Go/TS/JS/Rust)
#  ponytail: <the limit>, <what triggers the upgrade>     (#  for Python/Ruby/shell)
// ponytail: in-memory cache, swap for Redis if we run >1 instance
```

A shortcut with no named upgrade path is the kind that quietly becomes
permanent — name the trigger.

## What still earns a test

Non-trivial logic (a branch, a parser, a money/auth path) leaves one runnable
check behind — the smallest thing that fails if the logic breaks. Use the
project's test setup ([ fill in ]). No elaborate fixtures or per-function suites
unless asked. Trivial one-liners don't need a test.

---

## Worked examples — "use what the stack ships"

<!-- Keep the one that matches your stack, delete the rest. Each shows the
     precedence rules applied to a real decision, not just web. -->

**Web framework (e.g. Nuxt/Next/SvelteKit) — "add a date picker":**
not a date-picker _library_, not a hand-rolled calendar — `<input type="date">`
styled with our existing tokens, validated by the shared schema. Native, zero
deps, accessible. Upgrade later only if design needs range selection.

**Go — "we need a set of IDs":**
not a `Set[T]` generic package — `map[string]struct{}`, the idiom. Not a custom
error-matcher — `errors.Is`. Not a web framework for one endpoint —
`net/http`.

**Python — "build a lookup from two lists":**
not a loop that appends to a dict — `dict(zip(keys, values))`. Not a custom
LRU — `functools.lru_cache`. Not `requests` for one call in a script —
`urllib.request` (or the client we already vendor).

**Node (no framework) — "fetch JSON / make a UUID / clone an object":**
`fetch` (built in since 18), `crypto.randomUUID()`, `structuredClone()` —
before reaching for axios, the `uuid` package, or lodash.
