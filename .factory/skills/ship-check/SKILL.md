---
name: ship-check
description: Check whether the current project appears ready to ship. Use when the user asks for a pre-ship, release-readiness, or ship-readiness review.
---

# ShipCheck

## Purpose

ShipCheck will become a reusable pre-ship review workflow for checking a project's readiness before release, so a human can confirm it is safe to ship.

## Current Version

This is only the initial Skill scaffold. Detailed checks (tests, lint, build, Git, documentation, security, and release checks) will be added incrementally in later steps.

## Git State Check

Inspect the current repository using read-only Git commands only. Do not stage, discard, commit, stash, reset, or modify anything. Do not require a GitHub remote.

Determine:

1. the current branch,
2. whether tracked files have unstaged modifications,
3. whether anything is staged but not committed,
4. whether untracked files exist,
5. whether unresolved merge conflicts exist.

### Inspection approach

Use only read-only Git inspection commands:

- `git rev-parse --abbrev-ref HEAD` to get the current branch.
- `git status --porcelain` to detect staged, unstaged, and untracked changes in a stable, parseable format.
- `git diff --name-only --diff-filter=U` to detect unresolved merge conflicts (unmerged paths).

Interpret `git status --porcelain` output by its two-character status code:

- First column = staged status. A non-space first column means something is staged (e.g., `A`, `M`, `D`).
- Second column = unstaged status. A non-space second column means a tracked file has unstaged modifications (e.g., `M`, `D`).
- `??` = an untracked file.

### Result

PASS when all of the following are true:

- no staged changes,
- no unstaged tracked-file changes,
- no untracked files,
- and no unresolved merge conflicts.

FAIL when any of those conditions are present.

### Output

Output exactly the following block, using PASS or FAIL and Yes or No as appropriate:

```
Git State: PASS or FAIL
Branch: <branch>
Staged Changes: Yes or No
Unstaged Changes: Yes or No
Untracked Files: Yes or No
Merge Conflicts: Yes or No
Reason: <one short explanation>
```
