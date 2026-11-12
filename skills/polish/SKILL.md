---
name: polish
description: Polish and verify a feature or change after implementation. Use when the implementation is largely done and you want it shippable — runs /staff-review apply (covers comment hygiene + simplification), all project checks (tests, types, build, lint), and a docs sync check (JSDoc, README, docs site). Iterative-friendly — does not commit or open a PR by default. Pass "and open a pr" (or "commit", "ship", etc.) to also commit/push.
argument-hint: [optional follow-up instruction, e.g. "and open a pr", "commit", "and open a draft pr"]
---

# Polish a Change

Use this after writing feature code (or partway through) to get the change into shippable shape: lint clean, types clean, tests pass, docs in sync, and reviewed. Safe to invoke iteratively — by default it does not commit or open a PR.

## 1. Scope the change

Identify what changed. Look at:

- `git status` and `git diff` for uncommitted work
- `git diff $(git merge-base HEAD origin/main)...HEAD` (use `origin/master` if that is the base) for committed changes on this branch

If `git status` is empty (no uncommitted work), the entire change being polished is the branch-vs-base diff — read it in full, do not skip this step. A clean tree is not a signal that there is "nothing to review"; it is a signal that everything to review is already committed.

Read the actual diff contents critically, not just the file list. List the touched files and the public symbols (exported functions, classes, types) that were added, renamed, or changed signature. Note any behaviour changes worth flagging (new env vars, new flags, new external calls, dropped guards). Hold onto this list — later steps need it.

## 2. Verify

Detect the package manager from the lockfile (`yarn.lock` → yarn, `pnpm-lock.yaml` → pnpm, `package-lock.json` → npm). Then:

- Run the full test suite
- If a `tsc-check-tests` script (or equivalent) exists, run it when test files changed
- Run the build if package source changed
- Run lint

Fix anything that fails before continuing — there is no point reviewing broken code.

## 3. Verify docs are in sync

For every public symbol touched in step 1, verify documentation reflects the current code:

- **JSDoc**: each exported symbol has a doc comment, and signatures (params, return type, thrown errors, examples) match the current code. Add or update JSDoc where it is missing or stale. JSDoc on public APIs is documentation, not a code comment — the "drop if obvious" rule from `/staff-review`'s comment-hygiene lens does not apply here.
- **README**: grep README files for references to the touched symbols. Update examples that no longer compile or reference removed/renamed APIs.
- **Docs site**: if the repo has a `docs/`, `website/`, or similar, grep for the touched symbols there too and update any stale references.

Do not touch CHANGELOGs — they are generated, not edited by hand.

Do not add docs for private/internal symbols, and do not invent doc files that the project does not already use.

## 4. External review

Run `/staff-review apply` (`/b4nan:staff-review apply` when installed from the plugin). The `apply` token tells the skill to auto-apply surgical fixes for every verified finding that is a real improvement, regardless of severity, skipping only those whose fix is unnecessary or disproportionately complex. Do not run `/staff-review` without `apply` and re-implement the fixing yourself — that duplicates the skill's own apply branch. Comment hygiene and simplification opportunities are part of the staff-review surface — no separate manual pass needed.

## 5. Re-verify

After the review applied any fixes, re-run the relevant checks (tests, type-check, build, lint) on whatever was touched. If any check fails, fix and re-review.

## 6. Follow-up actions (parse `$ARGUMENTS`)

- **Empty / no clear follow-up instruction**: stop here. The polish is done; the user is iterating and will commit on their own schedule. Print a short summary of what changed during the polish (files touched by staff-review, docs updated, etc.).
- **Mentions commit only** (e.g. "commit", "and commit"): commit on the current branch using the project's git conventions — `feat:` for new functionality, `chore:` for non-functional changes, `fix:` only if the change is genuinely a bug fix. Do not push, do not open a PR.
- **Mentions PR / push / ship** (e.g. "and open a pr", "ship it", "push and open pr", "open a draft pr"): commit on the current branch, push it, and open a PR with `gh pr create` against the base branch. Honor modifiers in the instruction (e.g. "draft pr" → `--draft`).
- **Other instructions**: follow them. They may modify the above (e.g. "split into two commits", "rebase before pushing").

**Never commit or push to `master`/`main` directly.** If you are on `master`/`main` and the user asked for a PR, create a new branch off the base first and push that — even if the change is small or "obviously correct".

If you find yourself on `master`/`main` with changes already committed locally and the user asked for a PR, STOP — do not push. Move the commit to a new branch (`git branch <new-branch>` then `git reset --hard origin/master`) and push the branch instead. Ask the user before doing anything destructive to local state.
