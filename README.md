# Factory ShipCheck

ShipCheck is a reusable project-level Factory Skill for performing pre-ship readiness checks with Droid. It currently implements Git-state, automated-test, lint/static-analysis, and build readiness checks, and reports whether a project passes the current readiness criteria. The full release-readiness roadmap is not yet complete.

## Why I Built This

This project is being built while learning Factory/Droid incrementally. Each capability is added, tested, and documented as a separate step, so the reasoning behind every check is explicit and reviewable. The goal is to understand Factory Skills by building one, not to ship a finished product all at once.

## Current Capabilities

Four checks are implemented so far: the **Git State Check**, the **Test Check**, the **Lint Check**, and the **Build Check**.

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

### Lint Check

The Lint Check discovers an existing project-defined lint/static-analysis workflow, then runs the repository's own lint command. It:

- prefers explicit documented/default commands,
- inspects the underlying command or wrapper before execution,
- refuses commands that would mutate files,
- does not invent lint commands,
- does not install lint tooling,
- executes at most one safe, non-interactive, non-mutating command,
- and does not automatically fix lint findings.

The Lint Check has four possible results:

- **PASS** — a project-defined safe lint workflow was identified, executed, and exited 0.
- **FAIL** — a safe lint workflow was identified and executed, but returned nonzero due to lint/static-analysis findings.
- **NOT FOUND** — no project-defined lint/static-analysis workflow could be confidently identified.
- **BLOCKED** — the lint workflow could not safely or unambiguously be executed.

Two BLOCKED reasons we behaviorally tested:

1. The project-defined lint command is mutating.
2. Multiple equally valid lint commands exist and the project defines no default.

### Build Check

The Build Check discovers an existing project-defined local build workflow, then runs the repository's own build command in isolation. It:

- prefers explicit documented/default build commands,
- does not invent a build command,
- inspects the workflow for deploy/publish/release/remote side effects before execution,
- evaluates the CURRENT on-disk working-tree state rather than silently substituting HEAD,
- reproduces that current project state in a disposable workspace,
- executes the build only inside that disposable workspace,
- permits expected build artifacts to be generated there,
- removes the disposable workspace afterward,
- verifies the original repository remains unchanged,
- does not install missing dependencies or tooling,
- and treats the disposable workspace as project-tree filesystem isolation, NOT a full OS security sandbox.

The Build Check has four possible results:

- **PASS** — a project-defined build workflow was confidently identified, it was safely executed in a disposable workspace, and the build completed successfully.
- **FAIL** — the build workflow was identified, required local tooling was available, the build actually executed in isolation, and it returned nonzero because of a project/source/build error.
- **NOT FOUND** — no project-defined build workflow could be confidently identified.
- **BLOCKED** — a build workflow was identified but could not safely or meaningfully be executed.

Two BLOCKED reasons we behaviorally tested:

1. The documented build workflow included a remote artifact upload / publication side effect.
2. The documented build workflow required local build tooling that was unavailable.

A nonzero exit is not automatically FAIL. Missing tooling, dependencies, secrets, required services, or environment requirements can produce BLOCKED rather than FAIL.

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

The Lint Check outputs a block of the same shape. A passing result looks like:

```
Lint Check: PASS
Lint Command: sh lint.sh
Result: lint PASS: app.sh does not contain marker "LINT_ERROR"; exit status 0
Reason: The repository's documented non-mutating lint command completed successfully.
```

The Build Check outputs a block of the same shape. A passing result looks like:

```
Build Check: PASS
Build Command: sh build.sh
Isolation: disposable copy of current project state
Result: build completed successfully; dist/artifact.txt generated; exit status 0
Reason: The project-defined local build succeeded in isolation without modifying the original repository.
```

## How It Works

ShipCheck is a **project-level Factory Skill**. Its entry point lives at:

```
.factory/skills/ship-check/SKILL.md
```

- The Skill's frontmatter `description` helps Droid decide when the workflow applies (for example, when a user asks for a pre-ship or release-readiness review).
- The Skill body defines the actual readiness workflow: what to inspect, how to interpret it, and the PASS/FAIL criteria.
- The Git State Check uses only read-only Git inspection commands (`git rev-parse --abbrev-ref HEAD`, `git status --porcelain`, and `git diff --name-only --diff-filter=U`).

- The Test Check may execute at most one existing, project-defined, safe, non-interactive test command when the repository provides enough evidence to identify it confidently. It does not invent commands, install dependencies, or automatically fix failures.

- The Lint Check may execute at most one existing project-defined lint/static-analysis command, but only after inspecting the underlying workflow and establishing that it is non-mutating. It returns BLOCKED rather than executing a mutating or ambiguous workflow.

- The Build Check may execute at most one existing project-defined local build command. Because normal builds may intentionally generate files, ShipCheck first reproduces the current project state in a disposable workspace; the build executes only there; expected build artifacts may be created inside that workspace; the workspace is removed afterward; and the original repository is verified unchanged. Unsafe deploy/publish/upload workflows and unavailable required tooling return BLOCKED instead of being executed or misclassified. The disposable workspace is project-tree filesystem isolation, not a full OS sandbox.

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

### Lint Check

We performed a set of behavioral tests of the Lint Check:

1. **Factory ShipCheck repository with no lint workflow → NOT FOUND.**
   - No project-defined lint command existed.
   - ShipCheck did not invent or run a linter.
2. **Dependency-free lint demo with one documented canonical command `sh lint.sh` → PASS.**
   - ShipCheck inspected the underlying script before execution.
   - The script was established as read-only/non-mutating.
   - It executed and exited 0.
   - Lint Check → PASS.
3. **Same lint demo with exactly one temporary lint marker `# LINT_ERROR` → FAIL.**
   - The same lint command executed.
   - It reported the finding.
   - It exited 1.
   - Lint Check → FAIL.
4. **Exact committed `app.sh` restored → PASS again.**
   - The same lint command executed.
   - It exited 0.
   - Lint Check → PASS.
5. **Temporary project-defined mutating lint workflow `sh lint-fix.sh` → BLOCKED.**
   - ShipCheck identified the documented command.
   - ShipCheck inspected `lint-fix.sh` before execution.
   - The script contained an append redirect that would modify `app.sh`.
   - ShipCheck did not execute the command.
   - `app.sh` remained unchanged.
   - Lint Check → BLOCKED.
6. **Temporary ambiguity state with two equally valid, safe lint commands `sh lint.sh` and `sh lint-alt.sh` → BLOCKED.**
   - Both were independently verified as safe and passing.
   - The README explicitly defined neither as preferred or canonical.
   - ShipCheck executed neither command.
   - Lint Check → BLOCKED.

This demonstrates the Lint Check evaluates both the result of safe command execution and whether ShipCheck has enough safety and authority to execute a command in the first place.

### Build Check

We performed a set of behavioral tests of the Build Check:

1. **Factory ShipCheck repository with no build workflow → NOT FOUND.**
   - No project-defined build command existed.
   - ShipCheck did not invent a command.
   - No disposable workspace was created.
   - No build command was executed.
2. **Dependency-free build demo with one canonical command `sh build.sh` → PASS.**
   - The build intentionally creates `dist/artifact.txt` inside its current working directory.
   - ShipCheck identified the command from `README.md`, inspected `build.sh`, and determined it was a local build with relative workspace-local writes.
   - It reproduced the current project state in a disposable workspace, ran `sh build.sh` only inside that workspace, and observed `dist/artifact.txt` containing `ShipCheck build demo source`.
   - It observed exit status 0, removed the disposable workspace, and verified the original repository still had no `dist/`.
   - Build Check → PASS.
3. **Same build demo with `source.txt` deleted from the CURRENT working tree → FAIL.**
   - The committed HEAD still contained `source.txt`.
   - ShipCheck did NOT reconstruct `source.txt` from HEAD.
   - The disposable workspace represented the current on-disk state, so `source.txt` remained absent.
   - `sh build.sh` actually ran inside isolation; required shell tooling was available.
   - The build reported `source.txt` was missing and exited 1.
   - The failure was attributable to the current project state, not missing environment/tooling.
   - Build Check → FAIL.
4. **Exact committed `source.txt` restored → PASS again.**
   - The same build workflow ran in isolation.
   - `dist/artifact.txt` was generated in the disposable workspace.
   - Exit status 0; the disposable workspace was removed; the original repo remained clean.
   - Build Check → PASS.
5. **Temporary documented build workflow `sh build-release.sh` → BLOCKED.**
   - The wrapper contained `sh build.sh` followed by an outbound `curl` HTTP POST uploading `dist/artifact.txt`.
   - ShipCheck identified the documented build workflow and inspected the wrapper before execution.
   - It recognized the remote upload/publication side effect.
   - It did NOT create a disposable workspace, did NOT run `build-release.sh`, did NOT run `build.sh`, did NOT run `curl`, and made no network request.
   - Build Check → BLOCKED.
6. **Temporary documented build workflow `sh build-tooling.sh` → BLOCKED.**
   - The script required a deliberately unavailable local executable `shipcheck_demo_build_tool_9f3c7a`.
   - ShipCheck identified the documented workflow, inspected the required tooling, and checked availability with a read-only command lookup.
   - It confirmed the tool was unavailable, did NOT install or create the tool, did NOT execute the build workflow, and did NOT create a disposable workspace.
   - Build Check → BLOCKED.

These experiments demonstrate three separate Build Check responsibilities: discovering the project's real build workflow, safely isolating legitimate build-time filesystem mutations, and distinguishing project build failures from conditions where execution should be BLOCKED.

## Roadmap

The following checks are **planned** and will be added incrementally. They are not currently implemented:

- documentation
- security/configuration
- release readiness summary

## Built With

This project is being built and tested with Factory Droid. The initial development used GLM-5.2 (Droid Core), but the Skill does not require that specific model.

## Status

ShipCheck is an early work-in-progress. The Git State Check is implemented and behaviorally tested. The Test Check is implemented and behaviorally tested across PASS, FAIL, NOT FOUND, and BLOCKED. The Lint Check is implemented and behaviorally tested across PASS, FAIL, NOT FOUND, and BLOCKED. The Build Check is implemented and behaviorally tested across PASS, FAIL, NOT FOUND, and BLOCKED. The Build Check isolation was behaviorally tested by allowing build artifacts to be generated in a disposable workspace while the original repository remained clean. The Build Check BLOCKED behavior was tested for both remote publication/upload side effects and unavailable required tooling. Documentation, security/configuration, and release-readiness-summary checks are not yet implemented.
