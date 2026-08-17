---
name: ship-check
description: Check whether the current project appears ready to ship. Use when the user asks for a pre-ship, release-readiness, or ship-readiness review.
---

# ShipCheck

## Purpose

ShipCheck will become a reusable pre-ship review workflow for checking a project's readiness before release, so a human can confirm it is safe to ship.

## Current Version

ShipCheck currently implements the Git State Check, Test Check, Lint Check, Build Check, Documentation Check, and Security / Configuration Check. Release-summary checks are planned and will be added incrementally in later steps.

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

## Build Check

Inspect the repository for an existing project-defined build workflow, and only if a safe local build is confidently identified, run the repository's own build command in an isolated disposable workspace so the original repository is not modified. Do not invent a build command, do not install dependencies, do not deploy or publish, and do not alter the original working tree or Git history.

### Intent

Determine whether the project's existing build workflow succeeds before shipping, by discovering and executing the repository's own build command in isolation rather than assuming a command such as `npm run build` or `make build`.

### Discovery approach

Inspect the repository before executing anything. Prefer explicit project-defined or documented default build commands.

Possible evidence includes:

- package manager scripts (e.g., `package.json` `scripts.build`, with lockfiles indicating npm/pnpm/yarn),
- Makefile or task-runner/just targets (e.g., `make build`, `task build`, `just build`, `deno build`),
- Python build configuration (e.g., `pyproject.toml`, `setup.py`),
- Rust/Go/JVM/C/C++ project configuration (e.g., `Cargo.toml`, `go.mod`, `build.gradle`/`pom.xml`, `CMakeLists.txt`),
- repository documentation (e.g., README or AGENTS.md stating how to build),
- CI configuration (e.g., `.github/workflows`, `.circleci/config.yml`) as supporting evidence.

Configuration alone does not authorize inventing a command. Do not infer a runnable build command solely from the language/framework. Inspect wrapper definitions such as `npm run build`, `make build`, `task build`, and `just build` before executing. If multiple plausible commands exist with no project-defined default, return BLOCKED.

### Current-state definition

Build Check evaluates the CURRENT on-disk project state, not merely HEAD. The isolated workspace must represent:

- current tracked-file contents,
- staged and unstaged working-tree contents as they exist on disk,
- and relevant untracked files that are not ignored.

For partially staged files, use the current working-tree contents on disk. Do not silently substitute HEAD or an older committed version.

### Isolation strategy

First version: project-tree filesystem isolation in a disposable temporary directory.

1. Perform command discovery and safety inspection read-only in the original repository.
2. Create a unique disposable temporary directory outside the original repository.
3. For a Git repository, construct the disposable project tree from tracked files that currently exist on disk, plus untracked, non-ignored files.
4. Copy the CURRENT filesystem contents of those paths. Do not reconstruct tracked files from HEAD.
5. Do not copy `.git` metadata, ignored dependency directories, ignored caches, generated ignored artifacts, obvious local secrets/credentials, or other machine-local state merely to make the build succeed.
6. Preserve normal project directory structure and file modes where practical.
7. Symlink safety: inspect project symlinks before build execution. If a relevant symlink resolves outside the project root and safety cannot be confidently established, return BLOCKED.
8. Execute the build with its working directory inside the disposable workspace only.
9. Never intentionally run the project-defined build command from the original repository.
10. After execution, remove the disposable workspace.
11. Verify the original repository's Git status after cleanup and confirm ShipCheck did not alter it.

This is project-tree filesystem isolation, not a full operating-system sandbox. It does not prevent every possible process-level or home-directory side effect. If inspection indicates the build may write outside its working directory, use absolute paths, parent-directory writes, home-directory writes, or external infrastructure, return BLOCKED.

### Command and side-effect safety

Before executing a wrapper such as `npm run build`, `make build`, `task build`, or `just build`, inspect the underlying project-defined script or target. Do not assume the wrapper is safe merely because its name contains "build".

If the discovered command deploys, publishes, uploads artifacts remotely, pushes images, creates releases, or modifies remote infrastructure, return BLOCKED. Do not rewrite the project's build command into a different command. If inspection indicates the build may write outside the disposable workspace, return BLOCKED.

### Dependencies / secrets / network boundaries

- Do not install dependencies. Do not install runtimes or compilers. Do not copy ignored dependency directories merely to make the build pass.
- If required tooling or dependencies are unavailable in the disposable workspace, return BLOCKED. A missing dependency/tooling failure is NOT a project build FAIL.
- Do not intentionally provide secrets, credentials, signing keys, tokens, or local `.env` state to the isolated build. Do not run install/fetch/deploy/publish/release commands as Build Check.
- If the build requires secrets, credentials, required remote services, or unavailable network resources, return BLOCKED.

### Build vs deploy/release

A Build Check may run a local project build. It must NOT deploy, publish packages, push container images, upload artifacts to remote services, create releases, alter remote infrastructure, or substitute a deployment/release workflow for a local build. If the only project-defined build-like workflow includes those side effects and no safe local build is defined, return BLOCKED.

### Build / test overlap

If the project explicitly defines a safe local build command that internally performs tests or other local validation, use the project-defined build command as-is. Do not remove or rewrite those steps.

### Monorepos

Prefer one explicit repository-level default build workflow. If multiple independent build roots/workflows exist and no authoritative project-level default can be identified, return BLOCKED. A documented subdirectory build may be used only when the project explicitly identifies it as the workflow to run.

### Decision rules

1. Inspect the repository before executing anything.
2. Prefer explicit project-defined build commands or documented defaults.
3. Inspect wrapper scripts/targets before executing; do not assume a wrapper named "build" is a pure local build.
4. If the discovered command deploys, publishes, uploads, pushes images, creates releases, or performs remote side effects, return BLOCKED.
5. If multiple plausible build commands exist and there is no clear project-defined default, return BLOCKED.
6. If no build workflow can be confidently identified, return NOT FOUND.
7. If a workflow is identified but cannot be safely executed (missing runtime/tooling/dependencies, missing secrets/services, interactive/watch/server/indefinite, unsafe symlinks, possible out-of-workspace writes, or isolation cannot be established), return BLOCKED.
8. If a safe local build command is identified and isolation can be established, run at most that one command in the disposable workspace for this initial version.

### Exit classification

Exit status 0 ⇒ PASS.

For nonzero exits, distinguish:

- FAIL — the compiler/build/static generation actually ran, required local tooling was available, and the failure is attributable to project source/build errors.
- BLOCKED — required tooling/dependencies are missing, environment/secrets/network/services are unavailable, or ShipCheck cannot confidently distinguish the failure from an environment limitation.

Do not automatically classify every nonzero build exit as FAIL.

### Execution safety

- Execute at most one project-defined build command.
- Execute it only in the disposable workspace.
- Do not install anything.
- Do not modify the original repository.
- Do not stage, commit, or push anything.
- Do not deploy/publish/release/upload/push images.
- Do not rewrite the project's build command.
- Do not start interactive/watch/dev-server/indefinite processes.

### Results

PASS when all of the following are true:

- a project-defined build workflow was confidently identified,
- the current project state was safely reproduced in a disposable workspace,
- exactly one build command was executed inside that isolated workspace,
- and the build completed successfully.

FAIL when all of the following are true:

- a project-defined build workflow was confidently identified,
- the isolated workspace was created successfully,
- required local build tooling was available,
- the build command actually ran,
- and it returned nonzero because of a project/build failure attributable to the source or build itself.

NOT FOUND when:

- no existing project-defined or documented build workflow could be confidently identified.

BLOCKED when:

- a build workflow was identified, but ShipCheck could not safely or unambiguously execute it.

Examples of BLOCKED include:

- multiple plausible build commands with no clear project-defined default,
- required runtime/tooling/dependencies are unavailable,
- required ignored files, credentials, secrets, signing material, or external services are unavailable,
- the workflow requires installation or dependency fetching,
- the workflow is interactive, watch-mode, server-like, or indefinite,
- the workflow deploys, publishes, uploads artifacts remotely, pushes images, creates releases, or modifies remote infrastructure,
- the workflow appears capable of writing outside the disposable workspace,
- safe isolation of the current working-tree state cannot be established,
- unsafe external symlinks are present,
- or a nonzero exit is attributable to environment/tooling/dependency limitations rather than a project build error.

### Output

Output exactly the following block, using PASS, FAIL, NOT FOUND, or BLOCKED as appropriate:

```
Build Check: PASS, FAIL, NOT FOUND, or BLOCKED
Build Command: <command or none>
Isolation: <short isolation description or none>
Result: <short result>
Reason: <one short explanation>
```

## Documentation Check

Identify the repository's canonical release-facing documentation and determine whether its mechanically verifiable claims are consistent with the CURRENT repository state. This is not a "does README exist" check. Verify only explicit claims that can be grounded in concrete repository evidence. Read-only for this first version.

### Intent

Determine whether the project's canonical documentation is consistent with the current repository state, by verifying explicit mechanically verifiable claims rather than judging writing quality or completeness.

### Canonical documentation discovery

1. Prefer explicit repository/project configuration that identifies documentation authority.
2. Otherwise, a root `README.md` is the default canonical project documentation entry point.
3. Supporting documents may include `docs/`, `CONTRIBUTING.md`, `CHANGELOG.md`, `LICENSE`, and other explicitly referenced project documentation.
4. Supporting implementation evidence may include project configuration, manifests, scripts, and the project-level Factory Skill.
5. In a monorepo, prefer an explicitly documented/configured repository-level documentation root. If no explicit authority exists, root `README.md` remains the default. If genuinely competing canonical documentation roots exist and authority cannot be resolved, return BLOCKED rather than guessing.

### Current-state definition

Documentation Check evaluates CURRENT on-disk contents. If `README.md` or another canonical document has uncommitted edits, inspect those current contents. Do not silently substitute HEAD.

### Implementation evidence

A capability counts as concrete implementation evidence only when current repository structure explicitly defines it. Examples:

- an actual section/workflow/configuration implementing that named capability,
- a concrete project command/target/script,
- an existing documented entry-point file.

For ShipCheck specifically, an actual named check section in current `SKILL.md` is valid implementation evidence. Merely mentioning a capability in prose is not enough.

### Claim extraction

Inspect only atomic, explicit, mechanically verifiable claims. Strong candidates include claims containing:

- exact repository paths,
- exact local file names,
- exact canonical/default command names,
- exact capability names under Current Capabilities / Status / Roadmap,
- explicit words such as implemented, planned, not implemented, canonical, default, entry point,
- required relative repository links.

Do NOT treat every prose sentence or code block as a claim.

### Verification categories

1. **Local paths / entry points** — if canonical docs explicitly state that a path exists as a current project file, directory, Skill entry point, script, config, or other repository resource, verify that path in the current working tree. A missing required path is a FAIL finding.
2. **Canonical commands** — if docs explicitly identify a command as current/canonical/default, verify the corresponding project-defined script/target/manifest evidence exists. Do NOT execute the command. If command authority is ambiguous, return BLOCKED when necessary rather than guessing.
3. **Capability / status / roadmap consistency** — compare explicit current-capability/status/roadmap statements against clear implementation evidence. Only report a contradiction when both sides are explicit.
4. **Required relative documentation links** — verify required local relative links that point to repository files/directories. Do not fetch external HTTP/HTTPS links in this first version. Do not fail merely because an external link was not checked.
5. **Documented entry-point consistency** — paths such as `.factory/skills/...` should resolve when described as actual entry points.
6. **Negative claims** — verify only bounded, concrete negative claims when repository evidence can establish them confidently (e.g., "there is no package.json", "Build Check is not implemented"). Do not turn broad statements such as "there are no dependencies of any kind" into failures unless an explicit concrete contradiction is present.

### Capability / status / roadmap consistency

Only report a contradiction when both sides are explicit.

Examples:

- FAIL: README says "Lint Check is planned/not implemented" and current `SKILL.md` contains an actual implemented `## Lint Check`.
- FAIL: README says "Build Check is implemented" and no concrete Build Check implementation evidence exists.

A roadmap contradiction exists only when canonical docs explicitly identify a named capability as planned/not implemented AND current repository evidence explicitly implements that same named capability. Do not infer implementation merely from language/framework/config presence. Do not infer project intent. Do not fail merely because ShipCheck thinks a feature ought to be documented.

### Examples / future language / false-positive safety

Distinguish:

- examples from statements of current project state,
- optional commands from canonical/default commands,
- future/planned language from implemented/current language,
- illustrative code snippets from project-defined commands.

A code block alone is not a current canonical command unless surrounding documentation explicitly says it is.

Documentation Check must NOT judge grammar, style, writing quality, marketing language, subjective completeness, whether more documentation should exist, screenshot freshness, external-link uptime, or undocumented behavior merely because Droid believes it should be documented. Require concrete evidence before declaring FAIL.

### Generated documentation and authority

If the repository explicitly declares that checked-in documentation is generated and identifies a different authoritative source, use the declared source of truth when it is locally available and verifiable. If authority depends on unavailable tooling/private external sources or cannot be resolved confidently, return BLOCKED. Otherwise treat current checked-in canonical documentation as the documentation being verified.

### Decision rules

1. Inspect current repository state read-only.
2. Identify canonical documentation.
3. If none can be confidently identified, return NOT FOUND.
4. If documentation authority is genuinely ambiguous, return BLOCKED.
5. Extract only explicit verifiable claims within this first-version scope.
6. Gather concrete current repository evidence.
7. If a claim and evidence explicitly contradict one another, record a finding.
8. One or more confirmed findings, return FAIL.
9. No confirmed findings, return PASS.
10. When intent or authority must be guessed, return BLOCKED rather than FAIL.

### Execution safety

Documentation Check is read-only. Do NOT:

- modify documentation,
- automatically fix stale claims,
- execute test/lint/build commands,
- generate documentation,
- install documentation tooling,
- fetch external links for PASS/FAIL,
- stage, commit, or push anything.

### Results

PASS when all of the following are true:

- canonical project documentation was confidently identified,
- every mechanically verifiable claim within the first-version scope was consistent with current repository evidence,
- and no required local documentation references were broken.

FAIL when all of the following are true:

- canonical documentation was confidently identified,
- and at least one explicit mechanically verifiable documentation claim contradicts current repository evidence.

NOT FOUND when:

- no canonical project/release-facing documentation entry point could be confidently identified.

BLOCKED when:

- documentation exists, but ShipCheck cannot confidently determine the authoritative documentation set or safely verify the relevant claims without guessing.

### Findings format

For every FAIL finding, show a concise record underneath the top-level block:

```
Finding <n>:
Claim: <specific documentation claim>
Evidence: <specific repository evidence>
Issue: <short contradiction>
```

Do not overload the top-level block with every finding.

### Output

Output exactly the following top-level block, using PASS, FAIL, NOT FOUND, or BLOCKED as appropriate:

```
Documentation Check: PASS, FAIL, NOT FOUND, or BLOCKED
Documentation: <canonical documentation path(s) or none>
Issues: <count or unknown>
Result: <short result>
Reason: <one short explanation>
```

When there are findings, list them underneath the top-level block using the findings format above.

## Security / Configuration Check

Determine whether the CURRENT repository contains an explicit, mechanically detectable security/configuration artifact that represents a clear pre-ship risk within this check's deliberately narrow scope. This is not a general security audit, vulnerability scanner, penetration test, dependency CVE scanner, or architecture review. Read-only for this first version.

### Intent

Answer the narrow v1 question: does the current repository contain explicit, mechanically detectable security/configuration artifacts that represent a clear pre-ship risk within the check's defined scope? Prefer no finding over speculative detection.

### Current-state definition

Security / Configuration Check evaluates CURRENT on-disk repository state, including:

- tracked files,
- staged contents,
- unstaged tracked contents,
- non-ignored untracked files.

Do not silently substitute HEAD. A high-confidence finding in tracked, staged, or non-ignored untracked current project content is in scope. Do not assign different severity based only on whether the file is tracked, staged, or untracked in this first version. Record that state only as safe supporting metadata where useful.

### Ignored-file boundary

Do NOT inspect ignored files for secret contents. Ignored local `.env` files, credentials, caches, dependency trees, machine-local state, and generated local artifacts are outside this first version's repository-content inspection unless the repository explicitly promotes them into current project evidence. Their mere local existence must not produce FAIL.

### Discovery approach

Use a conservative read-only discovery sequence:

1. Inspect current Git/file state.
2. Determine ignored vs non-ignored project content.
3. Exclude `.git` internals.
4. Exclude ignored dependency/vendor/build/cache trees.
5. Inspect high-signal filenames and extensions.
6. Inspect explicit environment/configuration files.
7. Inspect supported structured credential formats.
8. Inspect for unmistakable private-key signatures.
9. Use explicit local repository security/configuration documentation as supporting evidence where relevant.

Do not recursively inspect ignored dependency/build/cache trees merely to search for secrets.

### High-confidence finding categories

Inspect only supported high-confidence finding categories.

1. **Private key material** — recognize unmistakable private-key blocks such as `BEGIN PRIVATE KEY`, `BEGIN RSA PRIVATE KEY`, `BEGIN OPENSSH PRIVATE KEY`, and equivalent unmistakable private-key structures. If such material appears in current in-scope project content, record a FAIL finding. Never print key contents. Never print the full matching line. Safe evidence wording example: `Evidence: PEM private-key header detected; contents redacted.`

2. **Structured credential files** — treat as findings only when filename/context AND structure strongly establish credential material. Supported first-version examples:
   - Google/service-account-style JSON: service-account structure is explicit and a `private_key` field contains private-key material.
   - AWS-style credential structure: an explicit access-key field paired with an explicit secret-access-key field, with values that are not empty or placeholders.
   Do not fail merely because a filename contains `secret`, `credential`, `token`, or `key`.

3. **Environment/configuration secret assignments** — inspect tracked/staged/non-ignored environment/config files where appropriate. A secret assignment finding requires a clearly sensitive field name AND a concrete non-placeholder value. Sensitive field semantics may include explicit secret/password/token/private-key/secret-access-key fields. Do not treat every configuration value as sensitive. Do not treat empty values, obvious placeholders, variable references, or template markers as confirmed secrets.

4. **Hard-coded secrets in ordinary source/config files** — be extremely conservative. A variable named `password`, `token`, `key`, or `secret` is not enough by itself. Do not use entropy-only guessing. Require strong structural/context evidence before reporting FAIL. For this first version, prefer no finding over speculative secret detection.

### Environment/config handling

A secret assignment in an environment/config file requires both a clearly sensitive field name and a concrete non-placeholder value. Do not treat every configuration value as sensitive. Distinguish real non-placeholder secret material from empty values, obvious placeholders, variable references, and template markers.

### Examples / templates / test fixtures / placeholders

Recognize obvious placeholder forms conservatively, including: empty values, `CHANGEME`, `PLACEHOLDER`, `YOUR_...`, `EXAMPLE`, `DUMMY`, `FAKE`, `TEST`, `<...>`, `${...}`, and repeated `x` / `XXXXX`-style placeholders. Case-insensitive where appropriate. Do not infer that every value outside this list is automatically a real secret.

Files clearly named as examples/templates, such as `.env.example`, `.env.template`, or sample configuration, should not produce ordinary env-secret findings for obvious fake/example values. However, an unmistakable private-key block or structurally concrete credential material is not automatically exempt merely because the path says example/template.

Do not fail on ordinary clearly fake test-fixture values. Do not automatically exempt unmistakable private-key material or structurally concrete credential material merely because a path includes `test`, `fixture`, `sample`, or `mock`. Require high confidence.

### Sensitive-file hygiene

Tracked, staged, and non-ignored untracked sensitive artifacts are all part of CURRENT repository state. If their contents meet a high-confidence finding rule, report FAIL. Do NOT fail merely because `.gitignore` is absent, a filename sounds sensitive, or an ignored local secret file exists.

### Configuration risks outside secrets

For this first version, EXCLUDE broad application-specific judgments such as `DEBUG=true`, permissive CORS, `0.0.0.0` binding, weak TLS configuration, development mode, container privilege, firewall settings, or application-specific auth policy, unless a future version defines a highly reliable project-agnostic rule. Do not produce speculative configuration findings.

### Symlink handling

Treat symlinks conservatively. Inspect the symlink path/target metadata. Follow only targets that resolve inside the repository. Never follow a target outside the repository. Never read arbitrary external filesystem content through a symlink. If an external or unresolved symlink is genuinely necessary to classify a relevant security artifact, return BLOCKED rather than guessing.

### Generated / vendor / binary boundaries

A file being generated does not automatically exempt it. Tracked/staged/non-ignored generated configuration may still be in scope unless it is inside an excluded build/cache/vendor area. Exclude common dependency/vendor/build/cache trees from content scanning in this first version. Do not inspect them merely to hunt for secrets. Do not content-scan binary blobs in this first version. A suspicious filename alone is not enough to FAIL. If a binary artifact is relevant but cannot be classified without unsafe or speculative inspection, do not guess.

### Git history

Current state only. Do NOT scan prior commits, deleted historical files, reflogs, or other Git history in this first version.

### Repository security-policy context

Explicit local repository security/configuration policy may be used as supporting context. It may clarify intended fixture/example files, authoritative configuration locations, and project-specific scope. It may not override an unmistakable private-key or concrete credential finding.

### False-positive safety

Distinguish:

- real credential material vs placeholders,
- current project files vs ignored local state,
- current config vs examples/templates,
- real credentials vs test/fake fixtures,
- private-key material vs documentation prose,
- code/documentation examples vs actual current credential artifacts.

A private-key phrase shown as plain documentation prose or a non-secret example must not automatically trigger a finding unless actual private-key structure/material is present. A README code block containing a placeholder secret must not automatically FAIL. No high-entropy guessing. No broad provider-token regex catalog in this first version.

### Secret-output safety

This is mandatory. If any sensitive material is detected, NEVER:

- print the secret value,
- print the private key,
- print the full sensitive line,
- print enough surrounding text to reconstruct the value,
- echo a credential merely to prove it exists.

Use only safe metadata. Use this finding format:

```
Finding <n>:
Type: <private key / credential / sensitive config / other supported type>
Path: <repository path>
Evidence: <safe redacted structural description>
Issue: <short explanation>
```

Examples:

- `Evidence: PEM private-key header detected; contents redacted.`
- `Evidence: non-placeholder AWS secret-access-key field detected alongside an access-key field; values redacted.`

### Decision rules

1. Inspect current repository state read-only.
2. Determine the non-ignored in-scope security/configuration surface.
3. If no meaningful v1 surface exists, return NOT FOUND.
4. Inspect only supported high-confidence finding categories.
5. Apply placeholder/example/fixture false-positive protections.
6. If one or more confirmed high-confidence findings exist, return FAIL.
7. If meaningful surface exists and no confirmed findings exist, return PASS.
8. If relevant evidence exists but cannot be safely or confidently classified, return BLOCKED.
9. Never expose secret values in output.
10. Never modify repository state.

### Execution safety

Security / Configuration Check v1 is read-only. Do NOT:

- modify files,
- delete exposed material,
- rotate credentials,
- add `.gitignore` entries,
- install scanners,
- execute Test/Lint/Build commands,
- invoke cloud CLIs,
- query external secret managers,
- fetch vulnerability databases,
- make network requests,
- stage, commit, or push.

### Scope exclusions

Explicitly exclude from this first version:

- ignored local files,
- `.git` internals,
- dependency/vendor trees,
- build artifacts and caches,
- entropy-only secret guessing,
- giant provider-token regex catalogs,
- binary content scanning,
- CVE/dependency vulnerability scanning,
- network security testing,
- runtime penetration testing,
- cloud-account inspection,
- external secret managers,
- Git history scanning,
- subjective security architecture review,
- broad application-specific configuration judgments.

### Results

PASS when all of the following are true:

- a meaningful v1 security/configuration surface was identified and inspectable,
- and no confirmed high-confidence finding within v1 scope was present.

FAIL when:

- at least one high-confidence, mechanically established security/configuration release-risk finding exists in current in-scope repository content.

NOT FOUND when:

- no meaningful security/configuration surface within v1 scope could be identified.

BLOCKED when:

- relevant security/configuration evidence exists, but ShipCheck cannot safely or confidently classify it without unavailable policy/tooling/context or guessing.

### Findings format

For every FAIL finding, show a concise record underneath the top-level block:

```
Finding <n>:
Type: <finding type>
Path: <repository path>
Evidence: <safe redacted evidence>
Issue: <short explanation>
```

No secret value may appear in output.

### Output

Output exactly the following top-level block, using PASS, FAIL, NOT FOUND, or BLOCKED as appropriate:

```
Security / Configuration Check: PASS, FAIL, NOT FOUND, or BLOCKED
Issues: <count or unknown>
Result: <short result>
Reason: <one short explanation>
```

When there are findings, list them underneath the top-level block using the findings format above.
