---
name: staff-review-inline
description: Wrapper around /staff-review that runs the review on a forked (clean) context but forwards the full final report inline to the parent conversation. Use this in UIs that don't surface forked-skill output (e.g. the macOS Claude app) where invoking /staff-review directly leaves you with no visible result.
argument-hint: "[pr-number-or-url] [apply]"
---

## Staff Review (Inline Output)

You are a thin pass-through. Your only job is to invoke the `staff-review` skill and forward its final output to the user verbatim.

The whole point of this wrapper: `/staff-review` uses `context: fork` so it gets a clean context for the review (important — do not change that). But some UIs do not render forked-skill output well, so you never see the report. This wrapper runs in the normal parent context and re-emits whatever the fork produced as plain assistant text, which every UI renders.

### Steps

1. **Pin the review target.** `staff-review` forks into a context whose working directory is the repo's **main checkout** — not yours. When the work under review lives in a git worktree, the fork would review the wrong branch, or (with no PR to go on) have to stop and ask. You are in the parent context and already know the answer, so pass it along:
   - Run `git rev-parse --show-toplevel`.
   - If it succeeds **and** the args below contain no `worktree=` / `cwd=` token, append `worktree=<toplevel>` to the args you forward in step 2. If the caller passed one explicitly, or you are not in a git repo, forward the args unchanged.

   This is scoping, not reviewing — that single command is the only git you may run.
2. **Invoke the `staff-review` skill** via the Skill tool (named `b4nan:staff-review` when installed from the plugin). Pass the arguments string below verbatim as the `args` parameter, plus any `worktree=` you appended in step 1.
   - `<ARGS>$ARGUMENTS</ARGS>`
   - If the markers are empty and step 1 found no repo, invoke `staff-review` with no `args` value.
3. **Wait for the skill to return** — it will run on its own forked context, do the full review (gather diff, identify issues, verify with subagents, re-evaluate severity, build the report, and apply fixes if `apply` was passed), and return a final report as the tool result.
4. **Forward that final report to the user verbatim** as your assistant text. Copy it exactly — every section, the findings table, per-finding detail, and the apply/skip summary. Do not paraphrase, summarize, truncate, or reformat.
5. **Do not add commentary** before or after the report. No "Here's the staff review:" preamble, no "Let me know what you want to do" trailer. Just the report.

### What NOT to do

- **Do not perform the review yourself.** Do not read the diff, do not grep the code, do not look at git history, do not verify findings, do not apply fixes. All of that happens inside the forked `staff-review` skill. You are only the relay. (The single `git rev-parse --show-toplevel` in step 1 is the sole exception — it scopes *where* the fork reviews, it does not review anything.)
- **Do not re-invoke `staff-review` if the result looks short.** A short report (e.g. "no findings") is a valid outcome — forward it as-is. The one exception is a result that is not a report at all: a bare preamble like "I'll start by…", an interim "once the agents return…", or anything with no findings section and no header. That means the fork died before reviewing. Retry it **once**; if the retry fails the same way, tell the user the fork produced no report and show what it returned — never pass a stub off as a review, and never fill the gap by reviewing yourself.
- **Do not invoke `staff-review-loop` instead.** This wrapper is for a single pass. If the user wants iterative review, they will invoke `staff-review-loop` directly (which has the same UI-rendering issue and would need its own wrapper).
