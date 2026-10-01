---
name: fix
description: Fix a GitHub issue or a bug described inline. Use when the user wants to fix a specific bug — by issue number, issue URL, or an inline bug description. Drives a test-first workflow with a surgical fix, multi-stage self-review, and a PR (never a direct push to main/master).
argument-hint: <issue number, URL, or bug description>
---

# Fix a Bug

## 1. Understand the problem

Parse `$ARGUMENTS` to determine what kind of input was provided:

- **GitHub issue number** (e.g. `123`) or **URL** (e.g. `https://github.com/…/issues/123`): fetch details with `gh issue view <number> --json title,body,labels,comments`. Read the issue thoroughly.
- **Bug description** (anything else, can be multiline): read it carefully as-is. This is the bug report — no need to fetch anything.

If the input includes a reproduction, understand exactly what fails and why.

## 2. Validate the report

Reproduce the behavior, then decide whether it is actually a defect. Not every report is one. Common non-bugs:

- The reporter's input means something different than they assume, and the code is handling it correctly.
- An underlying layer (runtime, platform, database, dependency) normalizes or transforms the input, and the difference the reporter sees is real.
- The expectation contradicts documented or intended behavior.

Verify the assumption the report rests on, independently of this codebase — read the spec or upstream docs, or exercise the underlying layer directly. If the code is faithfully reporting a real difference, there is no bug here.

If it turns out to be misusage or a wrong expectation: STOP. Do not write a test, do not touch source. Explain what actually happens, why, and what the reporter should change instead. A test written against a wrong expectation locks the wrong behavior into the codebase permanently, and every later step in this skill will treat it as the spec.

Before continuing, state the verdict in your reply, not just to yourself: either `real defect: <the wrong behavior you observed>` or `not a defect: <the reporter assumption that turned out to be false>`. Skipping this check quietly is easy; skipping a claim you have to write down is not. It also gives the user something to push back on before any code is touched.

## 3. Write a failing test

Once you have confirmed it is a real defect, and before touching any source code, write a test that reproduces the bug. Run it to confirm it fails with the reported error. This proves you understand the problem and gives you a regression guard.

## 4. Find the root cause

Use the failing test's stack trace and error message to locate the exact code path. Read the surrounding code to understand the design intent. Search for similar patterns in the codebase — the fix often becomes obvious once you find the right spot.

## 5. Fix surgically

**This is the most important step.** Apply the smallest possible change that fixes the bug:

- Look for ONE SMALL SPOT that needs adjusting rather than adding blocks of new code
- Reuse existing mechanisms — search for helpers, utilities, and patterns that already handle similar cases
- Prefer a 1-line fix over a 10-line fix. Prefer adjusting existing code over adding new branches
- Never add new parameters, methods, or abstractions unless absolutely unavoidable
- Never touch code outside the bug's scope — no cleanup, no refactoring, no "while I'm here" improvements

Ask yourself: "Am I adding code, or am I fixing the code that's already there?" If you're adding, try harder to find the adjustment.

## 6. Verify

Detect the package manager from the lockfile (`yarn.lock` → yarn, `pnpm-lock.yaml` → pnpm, `package-lock.json` → npm). Then:

- Run the specific test to confirm it passes
- Run the full test suite — do not skip this
- If a `tsc-check-tests` script exists in package.json, run it when test files changed
- Run the build if package source changed

## 7. Self-review

Before declaring done, critically review your diff:

- Is this the minimal fix? Could it be smaller?
- Am I reusing existing patterns or reinventing?
- Will this be easy to maintain, or am I making the codebase harder to understand?
- Did I leave any debug artifacts?

Then get an external review of the changes:

1. Run `/staff-review apply` (`/b4nan:staff-review apply` when installed from the plugin). The `apply` token tells the skill to auto-apply surgical fixes for every finding whose final severity is `medium`, `high`, or `critical`, and to skip / only report `low` / `nit` / `suggestion`. Do not run `/staff-review` without `apply` here and then re-implement the fixing yourself — that duplicates the skill's own apply branch. After it returns, re-run the relevant tests on whatever it touched. Comment hygiene and simplification opportunities are part of the staff-review surface — no separate manual pass needed.

Only proceed to commit once the review is clean (or remaining findings are explicitly low-severity).

## 8. Commit and open PR

**NEVER commit or push to `master`/`main` directly.** Always work on a dedicated branch and open a pull request — even if the current branch happens to be `master`/`main`, even if you have push permission, even if the fix is small or "obviously correct". No exceptions.

Workflow:

1. Create a new branch off the base branch with a descriptive name (e.g. `fix/7500-enum-namespace-functions`). Switch to it before staging anything.
2. Commit on that branch using the project's git conventions. Use `fix(<scope>):` prefix. If the input was a GitHub issue, add `Closes #<number>` in the commit body.
3. Push the branch (`git push -u origin <branch>`) — never push to `master`/`main`.
4. Open a PR with `gh pr create` against the base branch.

If you find yourself on `master`/`main` with the fix already committed locally, STOP — do not push. Move the commit to a new branch (`git branch <new-branch>` then `git reset --hard origin/master`) and push the branch instead. Ask the user before doing anything destructive to local state.
