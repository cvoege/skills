---
name: review-pr-stack
description: Trigger the claude github bot to review every PR in a stack. Use when someone asks for a review on an entire PR stack.

---

# Review the PR Stack

Re-trigger the Claude GitHub bot on every PR in the current `git stack`. For each
PR this hides the bot's previous review comments (so stale reviews don't clutter
the thread) and posts a fresh `@claude review once` comment to kick off a new
review.

**Requires**: `gh` authenticated, and the stack's parent pointers recorded (via
`git stack new` / `git stack parent`). The `git stack` utilities must be on PATH.

If any command in this series errors, or requires user intervention, pass control
back to the user.

## Process

### Step 0: Preconditions

```bash
git stack list                 # must print at least one branch; if it errors,
                               # the current branch isn't a tracked stack — stop
                               # and tell the user to run `git stack parent <parent>`
```

`git stack list` prints `<branch><TAB><parent>` lines, bottom → top. You only need
the first column (the branch names); order does not matter for reviews.

### Step 1: Walk the stack

For each `branch` from `git stack list`:

1. **Resolve the PR.** Fetch the PR number and its comments in one call:

   ```bash
   gh pr view <branch> --json number,url,state,comments
   ```

   If there is no PR for the branch (the command errors / returns nothing), note
   it and **skip** to the next branch — do not fail the whole run.

2. **Hide the bot's previous comments.** From the returned `comments`, take the
   `id` of every comment authored by `claude`. The `id` is already the
   GraphQL node ID (e.g. `IC_kwDODKw3uc7tpgEC`) — exactly what the minimize
   mutation needs:

   ```bash
   gh pr view <branch> --json comments \
     --jq '.comments[] | select(.author.login == "claude") | .id'
   ```

   For each such id, hide it as **outdated**:

   ```bash
   gh api graphql -f query='
     mutation($id: ID!) {
       minimizeComment(input: {classifier: OUTDATED, subjectId: $id}) {
         minimizedComment { isMinimized minimizedReason }
       }
     }' -F id="<node-id>"
   ```

   Re-minimizing an already-hidden comment is safe and idempotent. If a branch
   has no `claude` comments, skip this step and still do the next one.

3. **Trigger a new review:**

   ```bash
   gh pr comment <branch> --body "@claude review once"
   ```

Do these one branch at a time.

### Step 2: Report

Summarize per PR: how many `claude` comments were hidden and that a new
review was requested. List any branches skipped for lacking a PR.

## Critical rules

- **Skip branches with no PR.** Never fail the whole run because one branch lacks
  a PR — note it and move on.
- **Only touch `claude` comments.** Filter strictly on
  `author.login == "claude"`; never minimize a human's comment.
- **Hiding is best-effort and idempotent.** A comment that is already hidden is
  fine — don't treat it as an error.
- **Scope is top-level PR comments**, which is where the bot posts its review
  summary (and what `gh pr view --json comments` returns). Inline diff-review
  comments are out of scope.
- **Stop and hand back on any unexpected `gh`/GraphQL error.**
