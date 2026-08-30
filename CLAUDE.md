# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Two things:

1. **Standalone skills** at the repo root (`critical-review/`, `test-audit/`, `verify-work/`, `go-tutor/`, `study-mode/`), each a directory with a single `SKILL.md`. These are symlinked or copied into `~/.claude/skills/<name>/` for user-level use.
2. **[claude-boilerplate/](claude-boilerplate/CLAUDE.md)** — a copy-paste project scaffold. Its *contents* get pasted into a new or existing project's root: `.claude/skills/` (nine bundled skills), a steering `CLAUDE.md` (with `[ ... ]` stack slots to fill in), and a `.gitignore` carrying the `claude-artifacts/` entry. The bundled skills **live there canonically** — there are no duplicates elsewhere in this repo; editing them in place is editing the product.

There is no build, no test suite, no package manifest. The "code" is prompt content; edits ship by being read at invocation time.

When working *in this repo*, skills whose triggers match a prompt are directly executable: read the relevant `SKILL.md` and follow it as the procedure.

## Skill catalog — what exists and when to use it

**Verification suite** (repo root) — read-only, each writes a named report:

| Skill | Use when | Deliverable |
|---|---|---|
| [critical-review](critical-review/SKILL.md) | "review the branch", "code review", "critique the changes" | `review.md` (Blocker/Major/Minor/Nit) |
| [test-audit](test-audit/SKILL.md) | "are these tests good", "find coverage theater" | `test-review.md` (verdict + High/Med/Low) |
| [verify-work](verify-work/SKILL.md) | an implementing agent says "done"; "verify the work" | `verification-summary.md` (runs the two above in parallel) |

**Tutor modes** (repo root) — both refuse to hand over answers:

| Skill | Use when |
|---|---|
| [go-tutor](go-tutor/SKILL.md) | "tutor mode" for Go learning projects — hints and questions only, never writes the user's code |
| [study-mode](study-mode/SKILL.md) | "explain / forklar / help me understand", or "study mode" enabled — beginner-level explanation of decisions |

**Feature workflow** (in `claude-boilerplate/.claude/skills/`) — intent → spec → plan → build → docs:

| Skill | Use when | Deliverable |
|---|---|---|
| [feature-chain](claude-boilerplate/.claude/skills/feature-chain/SKILL.md) | "start a feature", planning any non-trivial build. Not for bugfixes/one-liners. | `claude-artifacts/<slug>/01-intent.md`, `02-spec.md`, `03-plan.md` (gitignored scratch, never committed) |
| [feature-docs](claude-boilerplate/.claude/skills/feature-docs/SKILL.md) | "feature is done", "wrap it up", "distil to docs" | `docs/<slug>.md` + HTML render — or the explicit verdict "nothing worth keeping" |
| [docs-as-static-html](claude-boilerplate/.claude/skills/docs-as-static-html/SKILL.md) | any standalone doc/runbook/handoff page; also the rendering step of feature-docs | self-contained `.html` in `docs/` |

**Code-minimalism suite** (in `claude-boilerplate/.claude/skills/`) — the ponytail family:

| Skill | Use when |
|---|---|
| [ponytail](claude-boilerplate/.claude/skills/ponytail/SKILL.md) | "be lazy", "yagni", over-engineering complaints — persistent mode, lite/full/ultra |
| ponytail-review / ponytail-audit | over-engineering check, diff-scoped / whole-repo |
| ponytail-debt | harvests `ponytail:` comments into a deferred-shortcut ledger |
| ponytail-gain / ponytail-help | scoreboard / reference card |

## Contracts between skills

Renaming a field, file, or bucket in one skill silently breaks its counterpart — check these when editing any member:

- **verify-work ↔ critical-review + test-audit**: verify-work is a thin orchestrator (parallel `general-purpose` sub-agents, single multi-tool-call message) that reads `review.md` severity buckets (Blocker/Major/Minor/Nit) and `test-review.md`'s verdict (`good tests` | `mixed` | `coverage-driven (gaps exist)`) + High/Med/Low counts verbatim. It deliberately duplicates no lenses, so edits to the two leaf skills propagate for free.
- **feature-chain ↔ feature-docs**: the handoff rides on `01-intent.md` frontmatter — `feature`, `branch`, `status: draft|building|shipped`. feature-docs locates the chain by matching `branch:` to the current git branch. Both SKILL.md files state the contract; keep them in sync.
- **feature-docs → docs-as-static-html**: feature docs are single-page (overriding that skill's multi-page default); markdown stays as source of truth beside the render; adding a page to an existing multi-page set requires re-embedding the rebuilt search index into **every** page.
- **ponytail → feature-docs**: `ponytail:` comments (shortcut + ceiling + upgrade trigger) are harvested from the feature diff into the doc's Known Limitations section. The comment format lives in ponytail's rules and the boilerplate `CLAUDE.md`; ponytail-debt reads the same markers.
- **claude-boilerplate/CLAUDE.md ↔ the bundled skills**: the boilerplate's steering file names the skills and their workflow rules (never commit `claude-artifacts/`, docs markdown is source of truth, proactive triggers). Renaming a bundled skill or changing its deliverable paths means updating that file too.

## Skill file contract

Every `SKILL.md` must begin with YAML frontmatter containing at minimum:

- `name:` — kebab-case slug matching the directory name.
- `description:` — one paragraph. This is what the harness shows to a model deciding whether to invoke the skill, so it must include both **what the skill does** and **the trigger phrases** that should invoke it ("Use when the user asks to …"). Treat the description as a routing prompt, not marketing copy. Include explicit **do-NOT-trigger** cases when adjacent behavior exists.

The body is freeform Markdown procedure. Conventions used across the existing skills:

- **Read-only posture is stated explicitly** when true ("does not modify code, run tests, or commit"). The deliverable is named up front.
- **Numbered `## Steps`** with shell snippets the skill expects to run. Default-branch discovery uses `git remote show origin | sed -n '/HEAD branch/s/.*: //p'` rather than hardcoding `main`.
- A **"What this skill is NOT"** or **"Hard rules"** section that fences off adjacent behaviors (e.g. critical-review does not fix findings; feature-docs does not re-review code).

## Authoring a new skill

1. Decide where it lives: repo root for standalone/user-level skills, `claude-boilerplate/.claude/skills/` if it belongs in the drop-in bundle.
2. Frontmatter `name` must equal the directory name.
3. Write the description with explicit trigger phrases — the harness routes on this text.
4. If the skill is read-only, say so and name the deliverable file(s) it writes.
5. Do not invent commands the user must run outside the skill — the procedure is the skill.
6. Add the skill to the catalog table above; bundled skills also get a row in the boilerplate `CLAUDE.md`'s skills table.
7. If it reads from or writes to another skill (report files, frontmatter fields, comment markers), document the contract in **both** SKILL.md files and in the contracts section above.
