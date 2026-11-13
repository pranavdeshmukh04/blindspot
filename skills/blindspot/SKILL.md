---
name: blindspot
description: Review a PR or branch with three agents and a judge. A blind planner writes the smallest plan for the requirement without seeing the diff; a bug reviewer and a bloat reviewer read the diff. Shows what the change missed, what it added beyond the requirement, which bugs to fix first, and how much shorter it could be. Use when the user runs /blindspot, or asks for a blind review, a second opinion on a PR, or whether a PR matches its ticket.
argument-hint: "[PR number] [--req <text | file | issue URL>]"
---

# Blindspot

Reviewers that read the diff first accept the author's approach. Blindspot
adds a planner that has never seen the change. It writes the smallest plan
that meets the requirement. Two more agents read the diff, one for bugs and
one for unnecessary code. You are the judge: you compare, merge, and report.

Run the steps in order. The whole run is read-only: never edit, commit, push,
or post comments anywhere.

Arguments: `$ARGUMENTS`

## Step 1: Collect inputs

1. Pick the change under review.
   - PR number given: `gh pr view <n> --json title,body,baseRefName,headRefOid,closingIssuesReferences`.
     Fetch the head if it is not local: `git fetch origin pull/<n>/head`.
   - No PR number: the current branch. If `gh pr view` finds a PR for it, use
     that PR's base. Otherwise the base is the repo's default branch.
2. Set `HEAD_SHA` and `BASE_SHA=$(git merge-base <base> <HEAD_SHA>)`.
   Stop with a one-line message if the two are equal.
3. Find the requirement. First match wins:
   1. `--req` argument: literal text, a file path, or an issue URL.
   2. An issue linked to the PR (`closingIssuesReferences`): `gh issue view`.
   3. A ticket key in the branch name or PR title, if a ticket connector
      (Jira, Linear, and so on) is available in this session.
   4. The PR description.
   5. None of these: ask the user for one sentence and wait.

   Record which source was used. Keep the requirement text exactly as found.
4. Create the blind worktree:
   ```bash
   BLIND_DIR=$(mktemp -d)/base
   git worktree add --detach "$BLIND_DIR" "$BASE_SHA"
   ```

## Step 2: Run three agents in parallel

Dispatch all three in one message so they run at the same time. Each gets
only its own prompt below with the placeholders filled in. Never pass this
conversation, and never pass one agent's output to another.

| Agent | Model | Gets |
|---|---|---|
| Blind planner | strongest available (`opus`) | `{REQUIREMENT}`, `{BLIND_DIR}` |
| Bug reviewer | mid-tier (`sonnet`) | `{REQUIREMENT}`, `{BASE_SHA}`, `{HEAD_SHA}` |
| Bloat reviewer | mid-tier (`sonnet`) | `{BASE_SHA}`, `{HEAD_SHA}` |

The planner must not receive the diff, the PR title or description (unless
the description is itself the requirement), the branch name, or `HEAD_SHA`.

### Blind planner prompt

```
You are planning a code change. You will not write the code.

## Requirement

{REQUIREMENT}

## Repository

The repository is checked out at {BLIND_DIR}. Work only inside that directory.
It is read-only for you: do not edit, create, or delete files.

Do not look at any other branch, commit, or pull request. Do not run
`git log --all`, `git branch`, `git fetch`, `git show` on another ref, or `gh`.
Someone may already have implemented this requirement elsewhere; your plan is
only useful if you have not seen that work.

Do not spawn subagents.

## Task

Write the smallest plan that fully meets the requirement in this repository.
For each thing the requirement asks for, stop at the first answer that holds:

1. Does it need to exist at all?
2. Does this repository already have it (a helper, type, or pattern)? Reuse it.
3. Does the language's standard library do it?
4. Does an already installed dependency do it?
5. Can it be one line?
6. Only then: the minimum new code.

The requirement says what the software must do. It does not list every input
or condition. Where it is silent, plan for what a reasonable user of this
software would expect, and record that as an assumption.

Read the code before planning. Cite real paths.

## Output

Return exactly these sections.

### Requirement detail
Two lines:
`Behaviour: clear` or `Behaviour: thin`, plus why. Is it clear what the
software must do?
`Placement: clear` or `Placement: thin`, plus why. Is it clear where in this
repository the change belongs?

### Requirements
Numbered list of what the change must do, in your own words. One line each.

### Plan
For each file to change or create: path, `edit` or `new`, what changes, and
which requirement number it serves. Name the functions and types to add.
Name existing code to reuse, with path.

### Edge cases
Inputs and conditions the change must handle. One line each.

### Size
Files touched, new files, new types, rough count of new non-test lines.

### Assumptions
Every gap you filled. One line each. `none` if there were none.
```

### Bug reviewer prompt

```
You are reviewing a code change for bugs. Nothing else.

## Requirement

{REQUIREMENT}

## Change

Base: {BASE_SHA}
Head: {HEAD_SHA}

Read it with:
  git diff --stat {BASE_SHA} {HEAD_SHA}
  git diff {BASE_SHA} {HEAD_SHA}
  git show {HEAD_SHA}:<path>        (a whole file as it is after the change)
  git grep -n <pattern> {HEAD_SHA}  (search the code after the change)

The working directory may be on a different commit. Trust only those commands.

If the repository has CLAUDE.md, AGENTS.md, or CONTRIBUTING.md at {HEAD_SHA},
read them first. Their rules apply to this review.

This review is read-only. Do not edit files, change branches, or commit.
Do not spawn subagents. If the diff is large, read it file by file and say so.

## What to look for

Code that does the wrong thing when it runs:
- crashes: nil or null access, out-of-range index, write to an unset map
- wrong results: off-by-one, wrong comparison, wrong unit, wrong default
- errors dropped, ignored, or returned without the caller being able to act
- unsafe concurrent access to shared state
- resources never released, loops that never end
- inputs the requirement names that the code does not handle
- security problems: injection, secrets in code, missing checks on input

Read enough of the surrounding code to be sure. A finding you cannot tie to a
line and a concrete failing input is not a finding.

Line numbers must be those of the file after the change. Take them from
`git show {HEAD_SHA}:<path> | cat -n` or `git grep -n`, never by counting
lines in the diff.

## Out of scope

Formatting, naming, comment style, and whether the code could be shorter or
simpler. Another reviewer covers those. Do not report them.

## Severity

- Critical: crash, data loss, security hole, or the feature does not work.
- Important: wrong result in a realistic case, or an error that is hidden.
- Minor: wrong only in an unlikely case, or a weak error message.

Not everything is Critical. If you are unsure between two levels, pick the
lower one.

## Output

Return exactly these sections.

### Findings
One block per finding, most severe first:

<Critical | Important | Minor>  <path>:<line>
What: what goes wrong, in one sentence.
When: the input or condition that triggers it.
Fix: the smallest change that fixes it.

`none` if you found nothing.

### Declined to judge
Things you looked at and set aside, one line each with the reason. Nothing
you set aside is dropped silently. `none` if there were none.
```

### Bloat reviewer prompt

```
You are reviewing a code change for unnecessary code. Nothing else.
The best outcome of this review is a shorter change.

## Change

Base: {BASE_SHA}
Head: {HEAD_SHA}

Read it with:
  git diff --stat {BASE_SHA} {HEAD_SHA}
  git diff {BASE_SHA} {HEAD_SHA}
  git show {HEAD_SHA}:<path>        (a whole file as it is after the change)
  git grep -n <pattern> {HEAD_SHA}  (search the code after the change)

The working directory may be on a different commit. Trust only those commands.

This review is read-only. Do not edit files, change branches, or commit.
Do not spawn subagents. If the diff is large, read it file by file and say so.

## What to look for

Only lines this change adds. For each, the kind of finding:

- unused: code nothing calls, fields nothing reads, options nothing sets
- standard library: hand-written code the language already ships. Name the
  function.
- already here: code that copies something this repository already has.
  Search before you claim it, and give the path of the existing code.
- not needed yet: an interface with one implementation, a factory with one
  product, a wrapper that only passes through, a layer with one caller
- shorter: the same logic in fewer lines. Show the shorter form.
- comment: a comment longer than the code needs, or one that repeats what the
  code already says

## Out of scope

Bugs, security, and performance. Another reviewer covers those. Do not report
them. Never report a test as unnecessary.

Do not report a line you would only rewrite to taste. Every finding must
remove lines: a finding that saves nothing is not a finding. Do not rewrite
code to fix a bug, and do not report a wrong value as unnecessary code.

Line numbers must be those of the file after the change. Take them from
`git show {HEAD_SHA}:<path> | cat -n` or `git grep -n`, never by counting
lines in the diff.

## Output

Return exactly these sections.

### Findings
One line per finding, biggest saving first:

<path>:<first line>-<last line>  <kind>: <what to cut>. <what replaces it>.  -<lines saved>

`none` if the change is already lean.

### Total
`-<N> lines possible out of <M> added.`
```

## Step 3: Judge

While the agents run, read the change yourself:

```bash
git diff --stat "$BASE_SHA" "$HEAD_SHA"
git diff --numstat "$BASE_SHA" "$HEAD_SHA"
git diff "$BASE_SHA" "$HEAD_SHA"
```

Leave generated files, lock files, and vendored code out of every count, and
list what you left out. If the diff is too large for one read, read it file
by file.

### 3a. Compare the plan with the change

Sort every difference into one of three groups.

**Asked for, but not found in the change.** A requirement or edge case from
the plan with no matching code in the diff. Before listing it, check the diff
and the base repo: it may be handled in a place the planner did not expect.
List it only if you looked and it is not there.

**In the change, but the minimal plan would not need it.** Files, types,
methods, options, layers, or dependencies in the diff that serve no numbered
requirement. Walk every new type and every new exported function and ask
which requirement it serves. This is a question for the author, not a defect.
The author may know something the requirement does not say. If the plan names
existing code that does the same job, give its path.

**Different approach.** Both meet the requirement in different ways. Mention
it only when the difference costs something concrete: more code, a new
dependency, or a missed reuse. Otherwise leave it out.

Do not report a difference you cannot tie to a file and line in the diff or to
a numbered requirement in the plan.

### 3b. Check the reviewers' findings

- Open the cited line for every Critical and Important bug finding. Drop a
  finding that is wrong and say so under "Dropped by the judge".
- Rule on every "Declined to judge" line: either raise it as a finding or
  leave it out.
- Check each "already here" bloat finding: the existing code must exist at
  the path given.

### 3c. Merge

The three sources overlap. Report each problem once.

- A missing requirement and a bug finding on the same lines are one item.
  Keep the bug finding's wording and add which requirement it breaks.
- A bloat finding on lines that are also "not needed by the minimal plan" is
  one item. Keep it in the requirement section with the lines saved.
- Must-fix items are: every missing requirement, every Critical finding, and
  every Important finding. Order them: crashes and missing requirements
  first, then by severity.

## Step 4: Clean up

```bash
git worktree remove --force "$BLIND_DIR"
```

Do this even if an earlier step failed.

## Step 5: Report

Print this layout as plain text in a code block. Keep the headings as written.
Write for a developer who has two minutes: say what to do, not only what is
wrong. No reviewer jargon. An empty section is one line, as shown.

```
BLINDSPOT   <PR number or branch>  <title>

VERDICT     <Ready to merge | Fix before merge | Needs rework>
            <N> must fix   <N> questions for you   <N> lines can go

------------------------------------------------------------
DO THESE FIRST
------------------------------------------------------------
1. <file:line>   <what goes wrong, and when>.
                 <Requirement it breaks, if any.>
                 Fix: <smallest fix>.

(or: nothing to fix)

------------------------------------------------------------
DOES THE CHANGE MATCH THE REQUIREMENT?
------------------------------------------------------------
Asked for, but not found in the change
  - <requirement>. (see <number above>)

In the change, but a minimal solution would not need it
(keep it if you have a reason, otherwise remove)
  - <file:lines>: <what it is>. <why the plan does not need it>.   -<lines>

Different approach
  - <file:line>: <what differs>. <what it costs>.

Size check
               Minimal plan    This change
  Files        <n>             <n>
  New lines    about <n>       <n>
  New types    <n>             <n>

------------------------------------------------------------
CAN THIS BE SHORTER?   <yes, by about N lines | no, already lean>
------------------------------------------------------------
  <file:lines>   <what to cut>. <what replaces it>.   -<lines>

------------------------------------------------------------
LOWER PRIORITY
------------------------------------------------------------
  <file:line>   <minor finding>. <fix>.

(or: nothing)

------------------------------------------------------------
HOW MUCH TO TRUST THIS REPORT
------------------------------------------------------------
  Requirement source   <--req | linked issue | ticket | PR description | asked>
  Behaviour            <clear | thin>: <planner's reason>
  Placement            <clear | thin>: <planner's reason>
  Planner assumed      <assumption, one per line, or none>
  Dropped by the judge <finding and why, one per line, or none>
  Not reviewed         <generated, lock, vendored paths, or nothing>
```

Verdict:
- `Needs rework`: three or more requirements are missing, or the change
  cannot work as written.
- `Fix before merge`: at least one must-fix item.
- `Ready to merge`: no must-fix items. Questions and shorter-code suggestions
  do not block.

"Lines can go" is the bloat total plus the lines of "not needed" items, each
line counted once.

If behaviour is `thin`, say so on the line under the counts. A thin
requirement makes every "missing" and "not needed" item less certain. Thin
placement alone does not weaken the report.

The size check is context, not a score. A change larger than the plan is
normal. Tests and error handling the planner did not count belong there.
