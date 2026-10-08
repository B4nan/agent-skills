# agent-skills

Agent skills I use day to day while maintaining [MikroORM](https://github.com/mikro-orm/mikro-orm) and other projects, packaged as a Claude Code plugin.

| Skill | What it does |
|-------|--------------|
| [`staff-review`](skills/staff-review/SKILL.md) | Deep review of a PR or branch diff. Verifies every finding (behavioral ones with a failing test it includes in the report), re-evaluates severity, and with `apply` fixes everything worth fixing. Runs in a forked context. |
| [`staff-review-loop`](skills/staff-review-loop/SKILL.md) | Runs `staff-review apply` repeatedly until the review comes back clean, stops converging, or hits the iteration cap (default 5). |
| [`staff-review-inline`](skills/staff-review-inline/SKILL.md) | Runs `staff-review` and relays its report as plain text, for UIs that don't show forked-skill output (e.g. the Claude desktop app). |
| [`fix`](skills/fix/SKILL.md) | Fixes a bug from an issue number, URL, or description: validates the report, writes a failing test first, applies a surgical fix, self-reviews, and opens a PR. |
| [`polish`](skills/polish/SKILL.md) | Gets a finished change shippable: runs all checks, syncs docs, and runs `staff-review apply`. Optionally commits and opens a PR. |
| [`triage-advisory`](skills/triage-advisory/SKILL.md) | Triages a GitHub security advisory: reproduces the claim, re-derives an honest severity, corrects the advisory fields, and drafts a reply to the reporter. |

## Installation

### Claude Code

```
/plugin marketplace add B4nan/agent-skills
/plugin install b4nan@b4nan-skills
```

Skills are namespaced by the plugin, so you invoke them as `/b4nan:staff-review`, `/b4nan:fix`, and so on.

### Other agents

Each skill is a plain `SKILL.md` following the [Agent Skills](https://agentskills.io) format, so you can also copy the `skills/<name>` directories into your agent's skills folder (e.g. `~/.claude/skills/`).

## Usage

```
/b4nan:staff-review                 # review the current branch
/b4nan:staff-review 123 apply       # review PR #123 and apply the fixes
/b4nan:staff-review-loop 3          # review + apply, up to 3 passes
/b4nan:fix 456                      # fix issue #456
/b4nan:polish and open a pr
/b4nan:triage-advisory GHSA-xxxx-xxxx-xxxx
```

## License

MIT
