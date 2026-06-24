---
name: breakout-pr-stack
description: Break out a single git branch into many PRs. Use when asked to split the work in one pull request into a stack of many.
---

# Break out a single git branch into many PRs. 

**Requires**: `gh` authenticated, and the stack's parent pointers recorded (via
`git stack new` / `git stack parent`). The `git stack` utilities
must be on PATH.


Lets organize the work in this branch. I want to split the work in many as-small-as-possible PRs. Each PR should be atomic, reviewable, and reasonably self-contained. Split by coherent changes, not coding steps. A stack should reflect how someone else should review the work, not the order you happened to write it. Each PR should land as one complete idea.

Propose a development flow in parts, so that each one fits a reasonable sized PR. Once we have built a plan, break out the changes into new branches using `git stack new` instead of `git checkout -b`. That will maintain a stack through the `git stack` utility. 

Once all the branches have been created, run the `create-pr-stack` skill to create all the PRs.
