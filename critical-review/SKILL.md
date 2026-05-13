---
name: critical-review
description: >
  Adversarial third-party code review of the currently checked-out branch vs the main
  branch. Produces a written `review.md` report with severity-tagged findings, anchored
  in file:line references. Read-only — does not modify code, run builds, or commit.
  Use when the user asks to "review the branch", "critique the changes", "code review",
  or "produce a review.md". The deliverable is the report; fixing findings is a separate
  task the user can request afterwards.
---

# Critical Review Skill

Act as a senior reviewer with no skin in the game. Go through the branch diff with a fine-toothed comb and produce a written report at `review.md` in the repo root.

This skill is **read-only**. Do not edit source files, run builds, run tests, or commit. The report is the deliverable — addressing findings is a separate task.

## Steps

### 1. Establish scope

```bash
git branch --show-current
git rev-parse --abbrev-ref HEAD
git remote show origin | sed -n '/HEAD branch/s/.*: //p'   # discover default branch
```

Use the discovered default branch (usually `main`, sometimes `master`/`develop`/`trunk`) as the base. If `git diff <base>..HEAD` is empty, stop and tell the user there's nothing to review.

```bash
git log <base>..HEAD --oneline
git diff <base>..HEAD --stat
```

**Cross-merge sanity check.** If the commit log shows commits with a different ticket prefix than the branch name (e.g. branch `feat/ABC-123-thing` but a `[FIX][XYZ-456]` commit appears), they likely arrived through a merge of another PR. Ask the user once whether to:

- **(a)** review only the branch's own ticket commits and add a **scope note** at the top of the report explaining what was excluded and why those files still appear in `git diff`; or
- **(b)** review the full diff.

Default to **(a)** if the user does not respond — out-of-scope work distorts the review and creates noise.

Once scope is fixed, list the in-scope commits explicitly in the report so the reviewee can audit your scoping.

### 2. Read the diff in full

```bash
git diff <base>..HEAD -- <in-scope paths>
```

For any file with non-trivial code changes, **read the full file** with the Read tool, not just the diff hunk. Bugs hide in the context above and below changed lines (a regex change is harmless until you see how the result is consumed three functions down).

For server/script/config files, also read sibling files they import from. A finding is only as good as your understanding of the surrounding code. If a Plan/Explore agent would let you cover ground faster on a large diff, use it — but you, the reviewer, must read the actual files for any finding you write down.

### 2.5. Pair changed files with their counterparts

Build a table before judging anything. For every changed production file, list:

| Prod file | Test file | Docs | Config / infra |
|---|---|---|---|
| `src/foo.ts` | `test/foo.test.ts` | `README.md` (section X) | `terraform/foo.tf` |

Rules:

- **Any prod change without a matching test change is a finding in its own right.** Severity depends on whether the prod change altered behavior (Major), added scaffolding (Minor), or was pure rename/format (Nit).
- A user-visible feature change without docs/CHANGELOG touched is a Minor finding by default.
- A schema/infra change without a migration or matching config update is a Major finding by default.
- If a test file *was* changed: in Step 4 you will check whether the test change actually exercises the prod change (not just touches the same area). For deep test-quality auditing, `test-audit` is the dedicated lens — this skill only flags the structural pairing gap.

This table goes into the report under the Scope section so the reviewee can audit your pairing.

### 3. Load project standards

If the repo has any of these, read them before judging:

- `CLAUDE.md` (root and any nested ones)
- `AGENTS.md`, `CONTRIBUTING.md`, `STYLE.md`, `.editorconfig`
- `README.md` (architecture and conventions sections)
- Any in-repo steering / convention docs (e.g. `docs/conventions/`, `.cursor/rules/`, `.kiro/steering/`)
- Spec/design docs matching the branch's ticket if discoverable

Skip docs the diff has no bearing on. Don't drown a docs-only branch in CDK rules.

### 4. Apply the review lens

Walk through every changed file by asking the questions below explicitly. A finding is only valid if you can answer "yes" with a specific file:line anchor and describe the concrete failure mode — the mutation that would slip past existing safeguards. Vague nits ("could be cleaner") are noise.

| # | Category | Question to answer |
|---|----------|--------------------|
| 1 | **Bugs** | If I inverted this conditional, deleted this line, or swapped this operator, would any caller or test fail? If no, the code path is unverified. |
| 2 | **Security** | What input could reach this sink — DB query, shell exec, file path, URL fetch, deserializer — without passing through the validation at line N? Trace the path explicitly. |
| 3 | **Backwards compatibility / deploy risk** | What in-production data shape, deployed environment, env var, or downstream consumer would break the first time this branch runs? Name the specific thing that breaks, not "could break things". |
| 4 | **Spec / requirements alignment** | Which stated acceptance criterion would fail if this branch shipped today, and which acceptance criterion was silently expanded beyond the ticket? |
| 5 | **Project conventions** | Which rule from the repo's own docs (Step 3) does this diff violate, and how does the violation actively mislead a future reader of this file? |
| 6 | **Test quality (structural only)** | Was a prod file changed without its test file being changed? Were `it(...)` blocks or assertions deleted or weakened in the diff? *For deep test-quality auditing — "would this test fail if the behavior regressed?" — defer to the `test-audit` skill rather than duplicating its lens here.* |
| 7 | **Code quality** | Is there dead code, leftover debug/TODO, magic numbers, or a comment that narrates WHAT (already in the identifier names) instead of WHY (the non-obvious constraint)? |
| 8 | **Documentation** | Is there a user-visible change with no matching docs/CHANGELOG/agent-doc update, or a removed feature still referenced in docs? |
| 9 | **Consistency** | Does style/naming/error-handling diverge from the surrounding code without justification, or is drive-by reformatting bundled with substantive edits in the same commit? |

For each finding, assign a severity:

- **Blocker** — ship-stopper. Data loss, security breach, broken build, breaking change with no migration plan.
- **Major** — must fix before merge. Real bug, exploitable weakness, convention violation that will mislead future readers.
- **Minor** — should fix. Code-quality, missing comment for non-obvious logic, test brittleness, doc drift.
- **Nit** — opinion. Take it or leave it. Don't pad the report with these.

**Severity defaults by row.** Bugs (Q1) and Security (Q2) failures default to Major; data-loss or remote-exploitable variants escalate to Blocker. Backwards-compat (Q3) defaults to Major when production data shape or live consumers are affected, Minor otherwise. Spec (Q4) and Conventions (Q5) failures default to Major when they materially change behavior or actively mislead future readers; otherwise Minor. Documentation (Q8) and Consistency (Q9) failures default to Minor unless they cause user-facing confusion. Q7 nits stay nits unless they hide an actual bug. Override these when a specific case warrants it — the defaults are a starting point, not a ceiling.

Be **critical but not punitive**. If the diff is good, say so — don't invent findings to look thorough. Equally, don't soften a real Major to a Minor to seem agreeable.

### 5. Write `review.md`

Write to `review.md` at the repo root using this structure:

```markdown
# Code Review — <branch-name>[ (<scope> only)]

**Reviewer:** Automated review pass (Claude)
**Base:** `<base-branch>`
**Head:** `<branch-name>`
**Scope:** <one-line scope statement>

---

## Scope note   <!-- only if cross-merged commits were excluded -->

<explain what was excluded and why those files still appear in git diff>

<list of in-scope commits>

### File pairing (from Step 2.5)

| Prod file | Test file | Docs | Config / infra |
|---|---|---|---|
| `<path>` | `<path or "—">` | `<path or "—">` | `<path or "—">` |

---

## Summary

<one paragraph: overall verdict, headline issues, top risks>

Severity legend: **Blocker** · **Major** · **Minor** · **Nit**

---

## <Area 1>   <!-- group by logical area, not by file -->

### `<file path>`

- **[Severity / Category]** <Specific finding with line numbers, anchored as `file.ts:42` or `file.ts:42-58`>. <Why it matters — name the concrete failure mode, not a vague risk>.
  **Suggested fix (pseudo-code, 5–15 lines):**

  ```
  // Pseudo-code preferred over runnable code — you have not executed
  // the suite, and a wrong runnable snippet is worse than a clear sketch.
  // Show the shape of the fix, not a copy-paste-ready patch.
  ```

(Repeat per file, per area. Do NOT include "things done well" bullets here — those go in the dedicated `What's solid` section below.)

---

## What's solid

<Mandatory section, even when short. List 2–5 areas where this diff genuinely got things right — design choices, test additions, well-handled edge cases, useful refactors. Without this section the report reads as pure negative and gets discounted by the reviewee. Be specific: `foo.ts:120 correctly guards against the empty-array case the previous implementation crashed on` beats `error handling looks good`.>

---

## Convention compliance

| Rule | Status | Notes |
|---|---|---|
| <rule from project docs> | ✅ / ⚠️ / N/A | <short note> |

---

## Deploy / operational notes

<Anything the reviewee needs to coordinate at merge/deploy time —
breaking renames, env var changes, follow-up tickets>

---

## Prioritised recommendations

1. **[Blocker]** <action>
2. **[Major]** <action>
3. **[Minor]** <action>
...
```

If `review.md` already exists, ask the user whether to overwrite, append, or write to a timestamped filename (`review-YYYY-MM-DD.md`).

### 6. Hand off

Print a one-paragraph summary to the user with the headline findings (Blocker/Major counts and one-line teasers) and a pointer to `review.md`. Do not edit code. Do not run builds or tests — the reviewee owns those.

If the user asks for fixes after seeing the report, that's a separate task — switch into normal implementation mode at that point and treat the report as the spec.

## Tone notes

- Anchor every claim in a file path and (where possible) line number. "`server.ts:90` accepts `name` from the URL without validating prefix" beats "the DELETE endpoint is unsafe".
- Prefer falsifiable statements. "`/g` flag is redundant on an anchored regex" beats "regex looks weird".
- When two interpretations are plausible, ask in the summary rather than guessing in a finding.
- Never recommend a "best practice" you can't justify from this codebase's conventions or a concrete failure mode in this diff.
- Don't grade the work. The report is for the implementer to act on, not a performance review.
