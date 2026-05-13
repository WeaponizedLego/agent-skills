---
name: test-audit
description: >
  Adversarial audit of test quality on the currently checked-out branch vs the default
  branch. Asks "would this test fail if the behavior regressed?", not "is coverage
  100%". Produces a written `test-review.md` with severity-tagged gaps and short
  pseudo-code sketches of the missing tests. Read-only — does not modify code, run
  tests, or commit. Use when the user asks to "audit the tests", "are these tests
  actually good", "review test quality", "find coverage theater", "would these tests
  catch a regression", or similar.
---

# Test Audit Skill

Act as a skeptical senior engineer reviewing tests written by someone else. The team already has 100% line coverage enforced — that is not the question. The question is whether the tests would actually **fail** if the production behavior regressed. Produce a written report at `test-review.md` in the repo root.

This skill is **read-only**. Do not edit source files, run builds, run tests, or commit. The report is the deliverable — writing the missing tests is a separate task the user can request afterwards.

## Steps

### 1. Establish scope

```bash
git branch --show-current
git remote show origin | sed -n '/HEAD branch/s/.*: //p'   # discover default branch
```

Use the discovered default branch (usually `main`, sometimes `master`/`develop`/`trunk`) as the base. If `git diff <base>..HEAD` is empty, stop and tell the user there's nothing to review.

```bash
git log <base>..HEAD --oneline
git diff <base>..HEAD --stat
```

**Cross-merge sanity check.** If the commit log shows commits with a different ticket prefix than the branch name (e.g. branch `feat/ABC-123-thing` but a `[FIX][XYZ-456]` commit appears), they likely arrived through a merge of another PR. Ask the user once whether to:

- **(a)** audit only the branch's own ticket commits and add a **scope note** at the top of the report explaining what was excluded; or
- **(b)** audit the full diff.

Default to **(a)** if the user does not respond.

**Pair up prod and test files.** Build a table of every changed production file and its corresponding test file(s). Any prod file with behavior changes but no matching test changes is itself a finding — note it.

If the diff has no test files at all, the report's only content is "no test changes for behavior changes in X, Y, Z" — write that report and stop.

### 2. Read both sides in full

For each changed production file: read the whole file with the Read tool, not just the hunk. The behavior under test is shaped by code above and below the diff.

For each changed test file: read the whole file. **Also** look at `git diff <base>..HEAD -- <test-file>` specifically for **deleted** `it(...)` blocks, removed assertions, or assertion weakening (`deep.equal` → `to.exist`, exact match → regex). A "refactor to fixtures" commit is the classic place where scenarios silently vanish.

For test fixtures: read them. A test that looks comprehensive may be reading from a fixture file that only covers half the input space.

### 3. Apply the test-quality lens

For each behavioral change in production, walk through these ten questions explicitly. Most findings come from Q1, Q5, and Q7 — those are the highest-yield checks, prioritize them when the diff is large.

| # | Question | What a failure looks like |
|---|----------|---------------------------|
| 1 | **Regression test.** If I deleted or inverted this prod line, would any test fail? | Test asserts on a derived value that's constant across fixtures → prod line could vanish and test passes. |
| 2 | **Asymmetric branches.** For every `if/else`, `??`, `?:`, early return, switch case — is each branch exercised? | One path has three tests, the other has none. |
| 3 | **Loose assertions.** Does the test verify *which* result came back, not just *that* a result came back? | `expect(result.length).to.equal(1)` without checking identity; `expect(x).to.exist`; partial-match regex when an exact match would do. |
| 4 | **Deep-equal coverage theater.** Are fields included in `deep.equal` fixtures but never read/sorted/mapped/filtered by production? | Field is required by type, set on insert, never consumed — only "covered" because it's in the expected object. |
| 5 | **Layered safety, single layer tested.** Does the production guarantee depend on layer A AND layer B, but tests only exercise A? | App-level dedup + DB unique index — only the dedup path is fed conflicting input. Drop the index, tests still pass. |
| 6 | **Non-determinism.** Uses `findOne` / Mongo cursor order / `Date.now()` / random IDs / Set/Map iteration — is the determinism source tested? | Non-unique index + `findOne`; tests pass because fixtures happen to insert in convenient order. |
| 7 | **Negative cases.** For every positive match test, is there a negative one verifying the *wrong* doc isn't returned? | Only positive matches tested; a too-permissive query change wouldn't break anything. |
| 8 | **Boundary/empty/edge inputs.** Empty array vs `undefined`, single-element vs many, min/max, locale variants, leap year, etc. | `visibleForDevices: []` early-return in prod, only `undefined` exercised in tests. |
| 9 | **Refactor regression.** Did the diff remove `it(...)` blocks or assertions without replacement? | "Refactor to fixtures" silently drops a scenario that previously caught a bug. |
| 10 | **Test-only smoke wrappers.** Handler/route tests that only assert status code while real logic is verified in a service test. | Handler test asserts `statusCode === 200` and empty body; no behavioral assertion at this layer. Note whether this is intentional layering or hiding gaps. |

### 4. Severity assignment

- **High** — Real behavior is unverifiable. A regression could ship silently. Includes Q1, Q5, Q6 failures; missing negative tests on auth/security/data-integrity paths.
- **Med** — Thin coverage. A plausible mistake during a future edit would not be caught. Includes Q2, Q4, Q8 failures.
- **Low** — Nits. Loose assertions where the tight form is trivial; redundant smoke wrappers; cosmetic test-name issues.

Do not invent findings to balance severities. If everything is fine, the verdict is "good tests" — say so.

### 5. Write `test-review.md`

At repo root, overwriting if it exists. Required structure:

```markdown
# Test audit — <branch name>

## Scope

- Base: <default branch>
- Commits in scope:
  - <sha> <subject>
  - ...
- Files reviewed: <list>
- Excluded (and why): <list, or "none">

## Verdict

**<good tests | mixed | coverage-driven (gaps exist)>**

<One paragraph. What's the overall shape of the suite? Where is it strongest, where is it weakest?>

## Findings

### <Area name, e.g. "Dashboard targeting">

- **[High] <one-line finding title>**
  - Production: [`path/to/file.ts:42-58`](path/to/file.ts#L42-L58)
  - Test: [`path/to/file.test.ts:120`](path/to/file.test.ts#L120) (or "no test exists")
  - **What's not verified:** <one sentence>
  - **Suggested test:**
    ```
    // pseudo-code, 5-15 lines, not runnable
    insert two generic-targeting docs for ('en','US',false) with no visibleForDevices
    expect the second insert to be rejected with a duplicate-key error
    ```

- **[Med] <next finding>**
  - ...

### <Next area>

...

## What's well-tested

- <Short list of areas where the suite is genuinely strong. This calibrates the report — without it the document reads as pure negative.>
```

### 6. Hard rules

- Read-only. The only file you may write is `test-review.md`. No `npm test`, no builds, no commits, no edits to source or tests.
- Every finding must have a `file:line` anchor for the production code and (where applicable) the test. No vague findings like "tests could be tighter".
- The "Suggested test" is **pseudo-code**, 5–15 lines max. Do not write runnable test code — you have not run the suite, and a wrong runnable test is worse than a clear sketch.
- One report per invocation. Overwrite the previous file.
- Do not grade on style, naming, or formatting. Those belong in a code review, not a test audit.
- Do not flag missing tests for code that was not changed on this branch. The audit is scoped to the diff.
- If a finding depends on production code being incorrect (not just untested), say so explicitly — but the recommended fix is still "write the test", not "change the code". A separate code review handles the latter.
