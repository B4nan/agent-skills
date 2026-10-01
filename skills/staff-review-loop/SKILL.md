---
name: staff-review-loop
description: "Run /staff-review iteratively, fixing all actionable findings each pass. Usage: /staff-review-loop [max_iterations]"
argument-hint: "[max iterations, default 5]"
---

# Staff Review Loop

Iteratively run `/staff-review apply` until the review is clean, the cap is reached, the loop stops converging, or a fix breaks the build. Each pass auto-applies surgical fixes for every verified finding that is a real improvement — regardless of severity — and skips only fixes the staff-review judges unnecessary or disproportionately complex for the severity.

**Run iterations back-to-back without pausing for user confirmation.** The whole point of this skill is to drive review → apply → re-review hands-off. Do not ask "should I continue?", "continuing?", "ready for iteration N+1?", or any equivalent between passes — the cap and the exit conditions below are the only stop signals. Waiting for a reply is not allowed.

**Relay every iteration's report — the user cannot see it otherwise.** `/staff-review` runs as a forked skill, and its report reaches you as a *tool result*, which many UIs never render. Assistant text written *between* tool calls is also not guaranteed to display until the turn ends — so if you emit one-liners mid-loop and save the detail for the end, an interrupted or long-running loop shows the user **nothing at all** (this has happened). Therefore, immediately after each `/staff-review` call returns, re-emit that iteration's full report (findings table, applied/skipped lists, validation summary) as plain assistant text before deciding whether to continue — and repeat the accumulated per-iteration summaries in the final message, which is the only text guaranteed to render everywhere.

## 1. Parse arguments

- `$ARGUMENTS` may contain a single positive integer — the maximum number of iterations. Default to **5** when empty or unparseable.
- Validate: must be an integer ≥ 1. If invalid, tell the user and stop.

Announce the cap to the user once before starting (e.g. `"Running staff-review loop, up to 5 iterations."`). Open a numbered list and pin the cap on the first line so it stays visible across long iterations:

```
Loop cap: <max_iterations>
1. Iteration 1: ...
2. Iteration 2: ...
```

Use the same list throughout — appending iteration entries instead of restarting numbering — so the count is always one glance away.

## 1.5. Pin the review target (do this ONCE, before the loop)

`/staff-review` runs in a **forked** context whose working directory is the repo's main checkout — it does **not** inherit your cwd. So when the work under review lives in a git worktree (or on any branch the main checkout doesn't have open), a bare `/staff-review apply` silently reviews the *wrong* branch. You are running here in the **non-forked** parent context, so detect the target now and pass it into every invocation.

From your current directory, capture:

- **Review directory** — `git rev-parse --show-toplevel`. This is the worktree/checkout the work actually lives in, and the only thing `/staff-review` strictly needs (it re-derives the base itself).
- **Branch** — `git branch --show-current` (used only for the sanity check below).
- **Open PR** — `gh pr view --json number` (empty if none; passed along as metadata).

**Sanity check before trusting it:** if the detected branch is a long-lived/default branch (`master`, `main`, `develop`, `v4`, …) with a **clean** tree and **no** open PR, you are almost certainly pointed at the wrong checkout. Run `git worktree list` and pick the worktree on the actual feature branch. If you still cannot tell which target is intended, **ask the user** rather than guessing — reviewing the wrong branch wastes a whole iteration.

Build a **scope suffix** from what you found and append it verbatim to the args of **every** `/staff-review apply` call below:

```
worktree=<absolute-toplevel-path>[ <pr-number-if-any>]
```

e.g. the full args become `apply worktree=/Users/me/proj/.claude/worktrees/feat-x 3851`.

## 2. Loop

Use an explicit counter `i`, starting at 1. **Before invoking iteration `i`, first check `i ≤ max_iterations`. If `i > max_iterations`, stop immediately with the cap-reached exit — do NOT invoke `/staff-review` again.** Treat the cap as a hard pre-condition on every iteration, not something to remember "at the end". The cap and the iteration counter must appear at the start of every iteration announcement (`"Iteration <i>/<max_iterations>:"`) so a one-line slip cannot lose track of where you are.

### 2a. Run the review with auto-apply

Announce `"Iteration <i>/<max_iterations>:"` and invoke the `/staff-review apply` skill (named `b4nan:staff-review` when installed from the plugin) **with the §1.5 scope suffix appended to its args** (`apply worktree=<dir> <pr>`), so it reviews and edits the target tree rather than whatever the forked checkout happens to have open. It will:

- Identify and verify findings.
- Re-evaluate severity post-verification.
- Apply the smallest surgical fix for every verified finding that is a real improvement, regardless of severity.
- Skip only findings whose fix is unnecessary (current code is fine as-is) or disproportionately complex for the severity, reporting the specific reason.
- Run the relevant tests / lint / build for the files it touched.

Capture the report: which findings were applied, which were skipped (with reason), and any test/build failures.

**Then relay it before moving on:** re-emit the iteration's full report as assistant text (per the relay rule at the top — the tool result itself is invisible to the user in many UIs). Only after relaying do you evaluate 2b.

### 2b. Decide whether to continue — automatically, no user input

Evaluate in this order:

1. **Zero applied findings this pass** → print `"Clean on iteration <i>. Stopping."` and break.
2. **Tests / build broke during the apply step** → print the failure and break. Do not iterate further.
3. **No progress** (this iteration's applied set — same file:line + same issue — matches the previous iteration's) → print `"Stuck on iteration <i>: same findings as last pass. Stopping."` and break.
4. **`i == max_iterations`** → print `"Cap reached on iteration <i>/<max_iterations>. Stopping."` and break. Do not invoke another `/staff-review`.
5. **Otherwise** → increment `i` and re-enter step 2a immediately. Do **not** ask the user whether to continue.

If the user interjects with a new instruction mid-loop, handle it, then resume the loop on the next iteration (or stop and report if the interjection redirects scope). Do not treat a normal interjection as implicit "stop the loop" — it just shifts what the next iteration sees.

## 3. Exit conditions

The loop exits when **any** of these happen — report which one triggered the exit:

1. **Clean review**: zero findings applied on the latest pass.
2. **Cap reached**: just completed iteration `max_iterations`. Stop **before** running an iteration past the cap, not after.
3. **No progress**: same applied set two iterations in a row.
4. **Fix failure**: tests / build broke and cannot be resolved.

## 4. Final report

At the end, output:

- Number of iterations run
- Which exit condition triggered
- Total findings applied across all iterations
- **The per-iteration reports (or faithful condensations keeping every finding + severity)** — interim text may never have rendered, so the final message must stand alone
- Any remaining skipped findings (for the user to review manually)
- Summary of files changed

Do not commit or push. Leave staging to the user.
