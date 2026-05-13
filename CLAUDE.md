# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A collection of Claude Code **skills** authored as standalone directories. Each subdirectory (`critical-review/`, `test-audit/`, `verify-work/`) contains a single `SKILL.md` — there is no build, no test suite, no package manifest. The "code" is prompt content; edits ship by being read at invocation time.

Skills here are designed to be symlinked or copied into `~/.claude/skills/<name>/SKILL.md` so the Claude Code harness can load them. References inside the skills (e.g. `~/.claude/skills/critical-review/SKILL.md` from [verify-work/SKILL.md](verify-work/SKILL.md)) assume that install location, not this repo path.

## Skill file contract

Every `SKILL.md` must begin with YAML frontmatter containing at minimum:

- `name:` — kebab-case slug matching the directory name.
- `description:` — one paragraph. This is what the harness shows to a model deciding whether to invoke the skill, so it must include both **what the skill does** and **the trigger phrases** that should invoke it ("Use when the user asks to …"). Treat the description as a routing prompt, not marketing copy.

The body is freeform Markdown procedure. Conventions used across the existing skills:

- **Read-only posture is stated explicitly** when true ("does not modify code, run tests, or commit"). The deliverable is named up front (e.g. `review.md`, `test-review.md`, `verification-summary.md` at the repo root).
- **Numbered `## Steps`** with shell snippets the skill expects to run. Default-branch discovery uses `git remote show origin | sed -n '/HEAD branch/s/.*: //p'` rather than hardcoding `main`.
- A **"What this skill is NOT"** or **"Hard rules"** section that fences off adjacent behaviors (e.g. critical-review does not fix findings; verify-work does not re-grade its sub-agents).

## Architectural relationship between the three skills

[verify-work/SKILL.md](verify-work/SKILL.md) is an **orchestrator** — it spawns `critical-review` and `test-audit` in parallel as `general-purpose` sub-agents via a single multi-tool-call message, then synthesizes their two report files into `verification-summary.md`. It is deliberately thin: it does not duplicate the lenses from the other two skills, so edits to `critical-review/SKILL.md` or `test-audit/SKILL.md` propagate automatically. When changing any of these three, check whether the contract between them (file names, return-report shape, severity buckets) still holds:

- `critical-review` returns Blocker / Major / Minor / Nit counts.
- `test-audit` returns a verdict (`good tests` | `mixed` | `coverage-driven (gaps exist)`) plus High / Med / Low counts.
- `verify-work` reads those exact fields verbatim — renaming a bucket in one skill silently breaks the synthesis in the other.

## Authoring a new skill

1. Create `new-skill-name/SKILL.md` at the repo root.
2. Frontmatter `name` must equal the directory name.
3. Write the description with explicit trigger phrases — the harness routes on this text.
4. If the skill is read-only, say so and name the deliverable file(s) it writes.
5. Do not invent commands the user must run outside the skill — the procedure is the skill.
