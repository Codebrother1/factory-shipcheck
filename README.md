# Factory ShipCheck

ShipCheck is a reusable project-level Factory Skill for performing pre-ship readiness checks with Droid. Its first implemented check inspects a repository's Git state and reports whether that state passes the current readiness criteria.

## Why I Built This

This project is being built while learning Factory/Droid incrementally. Each capability is added, tested, and documented as a separate step, so the reasoning behind every check is explicit and reviewable. The goal is to understand Factory Skills by building one, not to ship a finished product all at once.

## Current Capabilities

The only check implemented so far is the **Git State Check**. It inspects the current repository and reports:

- current branch
- staged changes (anything staged but not committed)
- unstaged tracked-file changes
- untracked files
- unresolved merge conflicts

The check is intentionally **read-only**: it inspects the repository using Git commands that cannot stage, discard, commit, stash, reset, or otherwise modify anything. A readiness check must never alter the state it is reporting on.

## Example Output

The ShipCheck Git State Check outputs a fixed, concise block. A passing result looks like:

```
Git State: PASS
Branch: main
Staged Changes: No
Unstaged Changes: No
Untracked Files: No
Merge Conflicts: No
Reason: Working tree is clean with no staged, unstaged, untracked, or conflicted changes.
```

A failing result, for example when an untracked file is present:

```
Git State: FAIL
Branch: main
Staged Changes: No
Unstaged Changes: No
Untracked Files: Yes
Merge Conflicts: No
Reason: Untracked files are present in the working tree.
```

## How It Works

ShipCheck is a **project-level Factory Skill**. Its entry point lives at:

```
.factory/skills/ship-check/SKILL.md
```

- The Skill's frontmatter `description` helps Droid decide when the workflow applies (for example, when a user asks for a pre-ship or release-readiness review).
- The Skill body defines the actual readiness workflow: what to inspect, how to interpret it, and the PASS/FAIL criteria.
- When the Skill applies, Droid uses only the read-only Git inspection commands currently defined by the Skill (`git rev-parse --abbrev-ref HEAD`, `git status --porcelain`, and `git diff --name-only --diff-filter=U`).

## Quickstart

This project has no package, no install command, and no dependencies. To try the Skill in a repository:

1. Copy the `ship-check` Skill directory into that repository at `.factory/skills/ship-check/` (so the entry point is `.factory/skills/ship-check/SKILL.md`).
2. Open that repository with Factory/Droid.
3. Ask for a pre-ship or release-readiness review (for example: "run a pre-ship readiness check on this repo").
4. Review the ShipCheck result block.

You can also invoke the Skill directly with `/ship-check`.

## What We Tested

We performed one behavioral test of the Git State Check against a real repository:

1. **Clean repository → PASS.** With a clean working tree, the check reported `Git State: PASS`.
2. **Add exactly one untracked temporary file → FAIL.** After creating a single untracked file (`shipcheck-test-untracked.txt`), the check reported `Git State: FAIL` with `Untracked Files: Yes`.
3. **Remove the temporary file → PASS.** After deleting that file, the check returned to `Git State: PASS`.

This demonstrated that the result changes in response to the actual repository state, rather than being a static or hardcoded outcome. The temporary file was not committed and was removed after the test.

## Roadmap

The following checks are **planned** and will be added incrementally. They are not currently implemented:

- tests
- lint
- build
- documentation
- security/configuration
- release readiness summary

## Built With

This project is being built and tested with Factory Droid. The initial development used GLM-5.2 (Droid Core), but the Skill does not require that specific model.

## Status

ShipCheck is an early work-in-progress. The Git State Check is implemented and tested; the other checks listed in the roadmap are not yet built.
