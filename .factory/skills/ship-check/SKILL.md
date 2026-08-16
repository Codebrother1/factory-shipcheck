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

## Test Check

Inspect the repository for an existing automated test workflow, and only if one is confidently identified, run the repository's own test command. Do not invent a test command, do not install dependencies, do not modify files, and do not fix failures.

### Intent

Determine whether the project's existing tests pass before shipping, by discovering and executing the repository's own test workflow rather than assuming a command such as `npm test`.

### Discovery approach

Inspect the repository before running anything. Prefer explicit project-defined evidence, and prefer a clearly documented/default test command when one exists.

Common evidence may include:

- package manager scripts and manifests (e.g., `package.json` `scripts.test`, lockfiles indicating npm/pnpm/yarn),
- Makefile or task-runner targets (e.g., `make test`, `task test`, `just test`, `deno task test`),
- Python project/test configuration (e.g., `pyproject.toml`, `setup.cfg`, `pytest.ini`, `tox.ini`, `conftest.py`),
- Rust, Go, JVM, or C/C++ project configuration (e.g., `Cargo.toml`, `go.mod`, `build.gradle`/`pom.xml`, `CMakeLists.txt`/CTest),
- repository documentation (e.g., README or AGENTS.md stating how tests are run).

CI configuration (e.g., `.github/workflows`, `.circleci/config.yml`) may be used as supporting evidence for identifying the intended test workflow, but do not blindly execute CI commands that may depend on CI-only secrets, services, containers, or environment setup.

### Decision rules

1. Inspect the repository before running anything.
2. Prefer explicit project-defined evidence.
3. Prefer a clearly documented/default test command when one exists.
4. If multiple plausible commands exist and there is no clear project-defined default, do not guess. Return BLOCKED.
5. If no automated test workflow can be confidently identified, return NOT FOUND.
6. If a workflow is identified but cannot be safely executed (see Execution safety), return BLOCKED.
7. If a safe, non-interactive existing test command is identified, run at most that one command for this initial version.

### Execution safety

- Do not install dependencies.
- Do not modify source files.
- Do not modify test files.
- Do not automatically fix failing tests.
- Do not start watch mode, a dev server, or an interactive/indefinite command.
- Run at most one identified test command for this initial version.
- Use the repository's existing workflow only.

### Results

PASS when all of the following are true:

- a test workflow was confidently identified,
- a safe, non-interactive existing test command was executed,
- and the test command completed successfully.

FAIL when all of the following are true:

- a test workflow was confidently identified,
- the test command was executed,
- and the command completed with a failure attributable to the tests.

NOT FOUND when:

- ShipCheck could not confidently identify an existing automated test workflow.

BLOCKED when:

- ShipCheck identified an intended test workflow, but could not safely execute it.

Examples of BLOCKED include:

- required dependencies are not installed,
- required runtime/tooling is unavailable,
- required secrets or external services are missing,
- multiple plausible test commands exist with no clear default,
- the only available command is interactive, watch-mode, server-like, or otherwise long-running.

### Output

Output exactly the following block, using PASS, FAIL, NOT FOUND, or BLOCKED as appropriate:

```
Test Check: PASS, FAIL, NOT FOUND, or BLOCKED
Test Command: <command or none>
Result: <short result>
Reason: <one short explanation>
```

## Lint Check

Inspect the repository for an existing project-defined lint/static-analysis workflow, and only if a safe, non-mutating one is confidently identified, run the repository's own lint command. Do not invent a lint command, do not install a linter, do not run a formatter or fixer, and do not modify files.

### Intent

Determine whether the project's existing lint/static-analysis workflow passes before shipping, by discovering and executing the repository's own lint command rather than assuming a command such as `eslint .` or `ruff check .`.

### Discovery approach

Inspect the repository before executing anything. Prefer explicit project-defined commands or documented defaults.

Possible evidence includes:

- `package.json` scripts (e.g., `scripts.lint`, with lockfiles indicating npm/pnpm/yarn),
- Makefile or task-runner/just targets (e.g., `make lint`, `task lint`, `just lint`, `deno lint`),
- `pyproject.toml` or related project configuration,
- ESLint/Biome/Ruff/ShellCheck/golangci-lint configuration,
- Rust/Go/JVM/C/C++ project configuration,
- repository documentation (e.g., README or AGENTS.md stating how lint is run),
- CI configuration (e.g., `.github/workflows`, `.circleci/config.yml`) as supporting evidence.

Configuration alone is supporting evidence only. Do NOT invent a command such as `eslint .` or `ruff check .` merely because a config file exists. Do not infer a lint workflow solely because a programming language is present. Do not automatically install a linter. If multiple plausible commands exist with no project-defined default, return BLOCKED.

### Lint vs type checking / formatting

- Do not independently classify a generic type-check command as the project's lint workflow.
- If the project explicitly defines a command as its lint workflow, ShipCheck may use it if the actual underlying command is safely non-mutating.
- Bare `tsc` or another potentially emitting compiler command is BLOCKED for this first version unless the project-defined command explicitly uses a non-emitting mode such as `--noEmit`.
- Do not treat a standalone formatting workflow as the Lint Check.
- A project-defined lint workflow may include a non-mutating formatting check, such as a check-only mode, if the entire workflow can be established as non-mutating.

### Command safety / mutation detection

Before executing a wrapper such as `npm run lint`, `make lint`, `task lint`, or `just lint`, inspect the underlying project-defined script or target. Do not assume the wrapper is safe merely because its name contains "lint".

If the underlying definition contains or invokes a modifying operation, return BLOCKED. Modifying operations include:

- `--fix`, `--fix-all`, `--write`, `--write-files`, `--apply`, `--apply-suggestions`, `--autofix`, `--auto-correct`, `--in-place`, or `-i`,
- auto-correct/apply behavior,
- a formatter in write mode (e.g., `prettier --write`, `ruff format`, `cargo fmt`, `gofmt -w`, `rustfmt`, `biome format --write`, `black`, `isort`),
- or another documented source-rewriting mode.

Do NOT remove mutating flags or rewrite the command into a safer version; that would invent a different project workflow. For ambiguous flags or opaque wrappers where mutation safety cannot be confidently established, return BLOCKED.

### Decision rules

1. Inspect the repository before executing anything.
2. Prefer explicit project-defined commands or documented defaults.
3. Distinguish lint/static analysis from formatting and auto-fix workflows; prefer the read-only lint form.
4. If the only discovered lint command uses `--fix`, `--write`, or another modifying mode, or is a formatter, return BLOCKED.
5. If multiple plausible lint commands exist and there is no clear project-defined default, do not guess. Return BLOCKED.
6. If no lint/static-analysis workflow can be confidently identified, return NOT FOUND.
7. If a workflow is identified but cannot be safely executed (see Execution safety), return BLOCKED.
8. If a safe, non-interactive, non-mutating existing lint command is identified, run at most that one command for this initial version.

### Execution safety

- Execute at most one project-defined lint command.
- Do not install tooling or dependencies.
- Do not modify source files or configuration.
- Do not automatically fix lint findings.
- Do not run a formatter as a replacement for lint.
- Do not stage, commit, or push anything.
- Do not start an interactive, watch, server, or indefinite process.

### Warnings and exit status

For this first version:

- exit status 0 ⇒ PASS,
- nonzero attributable to lint/static-analysis findings ⇒ FAIL.

Do not invent a warning threshold or reinterpret a project's configured exit behavior.

### Results

PASS when all of the following are true:

- a project-defined lint/static-analysis workflow was confidently identified,
- one safe, non-interactive, non-mutating existing lint command was executed,
- and the command completed successfully with exit status 0.

FAIL when all of the following are true:

- a project-defined lint/static-analysis workflow was confidently identified,
- the safe command was actually executed,
- and it returned nonzero due to lint/static-analysis findings.

NOT FOUND when:

- no existing project-defined or documented lint/static-analysis workflow could be confidently identified.

BLOCKED when:

- a lint workflow was identified, but ShipCheck could not safely or unambiguously execute one command.

Examples of BLOCKED include:

- the command or its underlying script/target uses `--fix`, `--write`, or another modifying mode,
- the command appears capable of rewriting source/config files,
- the only identified workflow is a formatter rather than a lint/static-analysis check,
- multiple plausible commands exist with no clear project-defined default,
- required tooling/runtime/dependencies are unavailable,
- the workflow is interactive, watch-mode, server-like, or indefinite,
- or ShipCheck cannot confidently establish that the command is non-mutating.

### Output

Output exactly the following block, using PASS, FAIL, NOT FOUND, or BLOCKED as appropriate:

```
Lint Check: PASS, FAIL, NOT FOUND, or BLOCKED
Lint Command: <command or none>
Result: <short result>
Reason: <one short explanation>
```
