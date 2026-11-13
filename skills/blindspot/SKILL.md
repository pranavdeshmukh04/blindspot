---
name: blindspot
description: Review a PR or branch against a minimal plan written by an agent that never sees the diff. Shows what the change missed and what it added beyond the requirement. Use when the user runs /blindspot, or asks for a blind review, a second opinion on a PR, or whether a PR matches its ticket.
argument-hint: "[PR number] [--req <text | file | issue URL>]"
---

# Blindspot

Reviewers that read the diff first accept the author's approach. Blindspot
starts from the other side: a planner that has never seen the change writes
the smallest plan that meets the requirement, and you (the judge) compare that
plan with what was actually built.

You are the judge. Run the steps in order. The whole run is read-only: never
edit, commit, push, or post comments anywhere.

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

## Step 2: Run the blind planner

Dispatch one subagent with the strongest model available (`opus`). Give it
the prompt below with `{REQUIREMENT}` and `{BLIND_DIR}` filled in, and nothing
else. Do not pass the diff, the PR title or description (unless the
description is itself the requirement), the branch name, `HEAD_SHA`, or any
part of this conversation.

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
`clear` or `thin`, plus one sentence saying why.

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

## Step 3: Compare

While the planner runs, read the change yourself:

```bash
git diff --stat "$BASE_SHA" "$HEAD_SHA"
git diff --numstat "$BASE_SHA" "$HEAD_SHA"
git diff "$BASE_SHA" "$HEAD_SHA"
```

Leave generated files, lock files, and vendored code out of every count, and
list what you left out. If the diff is too large for one read, read it file
by file.

When the plan comes back, sort every difference into one of three groups.

**Asked for, but not found in the change.** A requirement or edge case from
the plan with no matching code in the diff. Before listing it, check the diff
and the base repo: it may be handled in a place the planner did not expect.
List it only if you looked and it is not there.

**In the change, but the minimal plan would not need it.** Files, types,
options, layers, or dependencies in the diff that serve no numbered
requirement. This is a question for the author, not a defect. The author may
know something the requirement does not say. If the plan names existing code
that does the same job, give its path.

**Different approach.** Both meet the requirement in different ways. Mention
it only when the difference costs something concrete: more code, a new
dependency, or a missed reuse. Otherwise leave it out.

Do not report a difference you cannot tie to a file and line in the diff or to
a numbered requirement in the plan.

## Step 4: Clean up

```bash
git worktree remove --force "$BLIND_DIR"
```

Do this even if an earlier step failed.

## Step 5: Report

Print this layout as plain text in a code block. Keep the headings as written.
One or two lines per item. An empty group is a single line: `nothing found`.

```
BLINDSPOT   <PR number or branch>  <title>

VERDICT     <Matches the requirement | Check the questions below | Requirement not met>
            <N> missing   <N> questions for you

------------------------------------------------------------
DOES THE CHANGE MATCH THE REQUIREMENT?
------------------------------------------------------------
Asked for, but not found in the change
  - <requirement or edge case>. <where you looked>.

In the change, but a minimal solution would not need it
(keep it if you have a reason, otherwise remove)
  - <file:line>: <what it is>. <why the plan does not need it>.

Different approach
  - <file:line>: <what differs>. <what it costs>.

Size check
               Minimal plan    This change
  Files        <n>             <n>
  New lines    about <n>       <n>
  New types    <n>             <n>

------------------------------------------------------------
HOW MUCH TO TRUST THIS REPORT
------------------------------------------------------------
  Requirement source   <--req | linked issue | ticket | PR description | asked>
  Requirement detail   <clear | thin>: <planner's reason>
  Planner assumed      <assumption, one per line, or none>
  Not reviewed         <generated, lock, vendored paths, or nothing>
```

Verdict:
- `Requirement not met`: at least one item under "Asked for, but not found".
- `Check the questions below`: nothing missing, at least one question.
- `Matches the requirement`: both groups empty.

If the requirement detail is `thin`, say so on the line under the verdict.
A thin requirement makes every difference less certain.

The size check is context, not a score. A change larger than the plan is
normal. Tests and error handling the planner did not count belong there.
