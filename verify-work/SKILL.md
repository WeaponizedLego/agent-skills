---
name: verify-work
description: >
  Verification pass after another agent completes an implementation task. Runs the
  `critical-review` and `test-audit` skills in parallel against the currently
  checked-out branch and writes a combined `verification-summary.md` alongside the
  two underlying reports (`review.md`, `test-review.md`). Read-only — does not
  modify code, run tests, or commit. Use when an implementing agent says "done",
  or when the user asks to "verify the work", "check what was just done",
  "double-check the branch", or "run a verification pass".
---

# Verify Work Skill

You are the verification step that runs after another agent reports an implementation task as "done". You do not re-review the code yourself — you orchestrate two specialist reviewers in parallel and synthesize their findings.

This skill is **read-only**. The only files written are `review.md`, `test-review.md`, and `verification-summary.md`, all at the repo root. No source edits, no test runs, no commits.

## Steps

### 1. Sanity-check the diff exists

```bash
git branch --show-current
git remote show origin | sed -n '/HEAD branch/s/.*: //p'   # discover default branch
git diff <base>..HEAD --stat
```

If the diff is empty, stop and tell the user: "Nothing to verify — current branch has no changes against `<base>`." Do not spawn the sub-agents.

### 2. Run both skills in parallel

In a **single message**, spawn two `Agent` tool calls with `subagent_type: general-purpose`. They are read-only and write to different files, so parallel execution is safe and halves wall-clock time.

**Sub-agent A — code review:**

> Prompt:
> "Follow the procedure documented in `~/.claude/skills/critical-review/SKILL.md` exactly. Treat this prompt as the user invoking that skill against the currently checked-out branch. Produce `review.md` at the repo root, overwriting if it exists.
>
> When you are done, return a short report (under 200 words) containing:
> 1. The Blocker / Major / Minor / Nit counts.
> 2. The single headline finding (one line, with file:line anchor).
> 3. The list of in-scope commits you reviewed.
> 4. Anything that prevented you from completing the review."

**Sub-agent B — test audit:**

> Prompt:
> "Follow the procedure documented in `~/.claude/skills/test-audit/SKILL.md` exactly. Treat this prompt as the user invoking that skill against the currently checked-out branch. Produce `test-review.md` at the repo root, overwriting if it exists.
>
> When you are done, return a short report (under 200 words) containing:
> 1. The verdict line: `good tests`, `mixed`, or `coverage-driven (gaps exist)`.
> 2. The High / Med / Low counts.
> 3. The single headline finding (one line, with file:line anchor).
> 4. Anything that prevented you from completing the audit."

Do not paraphrase or shorten the underlying skill procedures inside this skill. The sub-agents read the SKILL.md files themselves — they are authoritative.

### 3. Synthesize `verification-summary.md`

After both sub-agents return, write `verification-summary.md` at the repo root using this structure:

```markdown
# Verification summary — <branch>

**Base:** `<default branch>`
**Verifier ran:** `critical-review` + `test-audit` (parallel)
**Date:** <ISO date>

## Headline

<One line. Examples:
- "2 Blocker, 3 Major code findings; test suite verdict: coverage-driven."
- "No code blockers; tests are good but miss the deleted-translation path."
- "Clean — ship as-is.">

## Code review

- Report: [`review.md`](review.md)
- Counts: **<N>** Blocker · **<N>** Major · **<N>** Minor · **<N>** Nit
- Top finding: <one line, with file:line anchor>

## Test audit

- Report: [`test-review.md`](test-review.md)
- Verdict: **<good tests | mixed | coverage-driven (gaps exist)>**
- Counts: **<N>** High · **<N>** Med · **<N>** Low
- Top finding: <one line, with file:line anchor>

## Recommended next step

<Exactly one of:
- "Address Blocker findings before merge."
- "Address Major findings, then re-run verify-work."
- "Add the <N> High-severity tests before merge."
- "Ship as-is — both reviews are clean.">
```

### 4. Hard rules

- **Read-only.** The only files you may write are `review.md` (via sub-agent A), `test-review.md` (via sub-agent B), and `verification-summary.md` (yourself). No source edits, no `npm test`, no commits.
- **Do not re-grade.** Severity counts, verdicts, and headline findings come verbatim from the sub-agents' returned reports. This skill synthesizes — it does not second-guess the underlying lenses.
- **Do not re-do the review work inside this skill.** If you find yourself reading prod source files or test files directly, stop — that's the sub-agents' job.
- **Failure tolerance.** If one sub-agent fails or returns an unusable result, still write `verification-summary.md` with the available half and an explicit `## Failures` section naming which report is missing and the error. Do not silently omit a half.
- **Overwrite.** All three files overwrite on each run. Do not preserve previous reports.
- **Parallel, not sequential.** Both Agent calls must go in a single message. If you find yourself spawning A, waiting, then spawning B, you've done it wrong.

### 5. Hand-off

After writing `verification-summary.md`, print a short message to the calling agent (or user):

> "Verification complete. <headline line>. See `verification-summary.md` for counts and pointers to the two underlying reports."

Do not include the full findings in the chat output — they live in the report files. The hand-off message is a pointer, not a replacement.

## What this skill is NOT

- **Not a trigger.** Invocation is left to the caller — a parent agent, the user, or a future hook configured in `CLAUDE.md` or `settings.json`. This skill does not configure its own automation.
- **Not a fixer.** If the user wants the findings addressed, that's a separate task invoked after they read the reports.
- **Not a re-implementation.** It does not duplicate the lenses from `critical-review` or `test-audit`. Edits to those skills propagate to verification automatically.
