# Factory ShipCheck

ShipCheck is a reusable project-level Factory Skill for performing pre-ship readiness checks with Droid. It currently implements Git-state and automated-test readiness checks, and reports whether a project passes the current readiness criteria. The full release-readiness roadmap is not yet complete.

## Why I Built This

This project is being built while learning Factory/Droid incrementally. Each capability is added, tested, and documented as a separate step, so the reasoning behind every check is explicit and reviewable. The goal is to understand Factory Skills by building one, not to ship a finished product all at once.

## Current Capabilities

Two checks are implemented so far: the **Git State Check** and the **Test Check**.

### Git State Check

The Git State Check inspects the current repository and reports:

- current branch
- staged changes (anything staged but not committed)
- unstaged tracked-file changes
- untracked files
- unresolved merge conflicts

The check is intentionally **read-only**: it inspects the repository using Git commands that cannot stage, discard, commit, stash, reset, or otherwise modify anything. A readiness check must never alter the state it is reporting on.

### Test Check

The Test Check inspects the repository to discover an existing project-defined automated test workflow, then runs the repository's own test command. It:

- prefers explicit documented/default commands,
- does not invent test commands,
- does not install dependencies,
- executes at most one safe, non-interactive existing test command,
- and does not automatically fix failures.

The Test Check has four possible results:

- **PASS** — a test workflow was confidently identified and the test command completed successfully.
- **FAIL** — the test workflow was identified and executed, but the tests failed.
- **NOT FOUND** — no existing automated test workflow could be confidently identified.
- **BLOCKED** — an intended test workflow was found, but ShipCheck could not safely or unambiguously execute one command.

Multiple equally valid test commands with no project-defined default produce **BLOCKED** rather than allowing Droid to guess.

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

The Test Check outputs a block of the same shape. A passing result looks like:

```
Test Check: PASS
Test Command: sh test.sh
Result: PASS: add 2 2 returned 4 (expected 4); exit status 0
Reason: The repository's documented canonical test command completed successfully.
```

The Test Check classifies a repository as one of **PASS**, **FAIL**, **NOT FOUND**, or **BLOCKED**, based on whether a test workflow was identified and, if so, whether the repository's own test command completed successfully, failed, or could not be safely executed.

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

### Git State Check

We performed one behavioral test of the Git State Check against a real repository:

1. **Clean repository → PASS.** With a clean working tree, the check reported `Git State: PASS`.
2. **Add exactly one untracked temporary file → FAIL.** After creating a single untracked file (`shipcheck-test-untracked.txt`), the check reported `Git State: FAIL` with `Untracked Files: Yes`.
3. **Remove the temporary file → PASS.** After deleting that file, the check returned to `Git State: PASS`.

This demonstrated that the result changes in response to the actual repository state, rather than being a static or hardcoded outcome. The temporary file was not committed and was removed after the test.

### Test Check

We performed a set of behavioral tests of the Test Check:

1. **Factory ShipCheck repository with no test workflow → NOT FOUND.** With no project-defined test command, the check reported `Test Check: NOT FOUND`.
2. **Dependency-free demo repository with a single documented canonical command `sh test.sh` → PASS.** The command executed, exited 0, and the check reported `Test Check: PASS`.
3. **Same demo repository with exactly one intentional application bug → FAIL.** The same test command executed, the assertion failed, exited 1, and the check reported `Test Check: FAIL`.
4. **Exact committed application file restored → PASS again.** The same command executed, exited 0, and the check reported `Test Check: PASS`.
5. **Demo repository temporarily documented two equally valid test commands with explicitly no default → BLOCKED.** ShipCheck did not choose either command and executed no test command, reporting `Test Check: BLOCKED`.

This demonstrates the Test Check responds to repository evidence and actual command execution rather than producing a static classification.

## Roadmap

The following checks are **planned** and will be added incrementally. They are not currently implemented:

- lint
- build
- documentation
- security/configuration
- release readiness summary

## Built With

This project is being built and tested with Factory Droid. The initial development used GLM-5.2 (Droid Core), but the Skill does not require that specific model.

## Status

ShipCheck is an early work-in-progress. The Git State Check is implemented and behaviorally tested. The Test Check is implemented and behaviorally tested across PASS, FAIL, NOT FOUND, and BLOCKED. The remaining roadmap checks are not yet implemented.
