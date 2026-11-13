# Blindspot

A Claude Code plugin that reviews a pull request from the side most reviewers skip.

Most AI reviewers read your diff first, so they accept your approach and comment on details. Blindspot runs a planner that never sees your change. It reads only the requirement and the code as it was before, and writes the smallest plan that would solve it. A judge then compares that plan with what you built and tells you:

- what the requirement asked for that the change does not do
- what the change adds that a minimal solution would not need

## Install

```
/plugin marketplace add pranavdeshmukh04/blindspot
/plugin install blindspot@blindspot
```

## Use

```
/blindspot                     review the current branch
/blindspot 42                  review pull request 42
/blindspot --req "add rate limiting to the orders API"
```

The requirement is taken from `--req`, a linked issue, a ticket, or the PR description, in that order. If none is found, Blindspot asks you for one sentence.

Blindspot only reads. It never edits code, commits, or posts comments.

## Layout

```
.claude-plugin/     plugin manifest and marketplace file
skills/blindspot/   the skill
```

## Status

Early version. Bug review, bloat review, and test coverage ranking are planned.

## License

MIT
