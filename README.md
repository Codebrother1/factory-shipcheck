# Factory ShipCheck

ShipCheck is a reusable project-level Factory Skill for performing pre-ship readiness checks with Droid. It currently implements six readiness checks (Git State, Test, Lint, Build, Documentation, and Security / Configuration) plus the Release Readiness Summary orchestrator, and reports whether a project passes the current readiness criteria. The current ShipCheck readiness roadmap is implemented.

## Why I Built This

This project is being built while learning Factory/Droid incrementally. Each capability is added, tested, and documented as a separate step, so the reasoning behind every check is explicit and reviewable. The goal is to understand Factory Skills by building one, not to ship a finished product all at once.

## Current Capabilities

Six readiness checks are implemented, plus the **Release Readiness Summary** orchestrator that aggregates them: the **Git State Check**, the **Test Check**, the **Lint Check**, the **Build Check**, the **Documentation Check**, and the **Security / Configuration Check**.

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

### Documentation Check

The Documentation Check identifies the repository's canonical release-facing documentation and verifies that its mechanically verifiable claims are consistent with the current repository state. It:

- identifies canonical project documentation,
- defaults to root `README.md` when no stronger project-defined authority exists,
- evaluates CURRENT on-disk documentation rather than silently substituting HEAD,
- verifies only explicit mechanically verifiable claims,
- uses concrete repository evidence,
- verifies current local paths / entry points,
- verifies explicitly canonical/default commands without executing them,
- checks explicit capability / Status / Roadmap consistency,
- distinguishes examples from current project claims,
- distinguishes planned/future language from implemented/current language,
- does not judge grammar, writing quality, marketing language, or subjective completeness,
- does not fetch external links for this first version,
- does not automatically fix documentation,
- and is read-only.

The Documentation Check has four possible results:

- **PASS** — canonical documentation was identified and all mechanically verifiable claims within scope were consistent.
- **FAIL** — canonical documentation was identified and one or more explicit mechanically verifiable claims contradicted current repository evidence.
- **NOT FOUND** — no canonical project/release-facing documentation entry point could be confidently identified.
- **BLOCKED** — documentation exists but ShipCheck cannot confidently determine documentation authority or verify the relevant claims without guessing.

All four Documentation Check outcomes have now been behaviorally exercised.

### Security / Configuration Check

The Security / Configuration Check examines the CURRENT non-ignored repository state for a deliberately narrow set of high-confidence security/configuration release risks. Current in-scope state may include tracked files, staged contents, unstaged tracked contents, and non-ignored untracked project content. It:

- respects ignore boundaries,
- does not inspect ignored local secret files,
- does not silently substitute HEAD,
- does not scan Git history,
- does not install scanners,
- does not query external secret managers,
- does not fetch vulnerability databases,
- makes no network requests,
- does not perform general vulnerability/CVE scanning,
- does not perform penetration testing,
- does not make broad application-specific security judgments,
- and is read-only.

The narrow high-confidence v1 finding categories are:

- unmistakable private-key material,
- narrowly supported structured credential files,
- concrete non-placeholder sensitive environment/configuration assignments,
- hard-coded credentials only when strong structural/context evidence exists.

Entropy-only guessing is excluded. A filename containing "secret", "token", "credential", or "key" is not enough by itself. Obvious placeholders and variable references are not findings. Example/template/test-fixture context is handled conservatively.

#### Secret output safety

If sensitive material is detected, ShipCheck must NEVER print:

- the secret value,
- private-key contents,
- the full sensitive line,
- or enough surrounding text to reconstruct the value.

Findings use safe redacted structural metadata only.

The Security / Configuration Check has four possible results:

- **PASS** — meaningful in-scope security/configuration surface was identified and inspected, and no confirmed high-confidence finding was present.
- **FAIL** — at least one high-confidence, mechanically established current release-risk finding was present.
- **NOT FOUND** — no meaningful security/configuration surface within v1 scope was identified.
- **BLOCKED** — relevant security/configuration evidence exists, but ShipCheck cannot safely or confidently classify it without crossing a forbidden/unavailable boundary or guessing.

### Release Readiness Summary

The Release Readiness Summary is an orchestrator over the six existing checks, not a seventh independent repository check. It:

- runs the six checks in their Skill-defined order,
- preserves each check's native result exactly,
- does not invent a seventh inspection domain,
- does not weaken any underlying check's safety rules,
- runs all six by default unless an orchestration-safety problem makes later results unreliable,
- verifies original-repository state integrity between checks,
- does not auto-reset/stash/restore unexpected mutations,
- preserves Security / Configuration redaction boundaries,
- and performs no release actions.

The Release Readiness Summary has four possible results:

- **READY** — all six underlying checks returned PASS. READY means ready within ShipCheck's current six-check readiness scope, not universally risk-free.
- **NOT READY** — one or more underlying checks returned FAIL.
- **BLOCKED** — no FAIL exists, but one or more checks returned BLOCKED, or an orchestration-safety problem prevents reliable completion.
- **INCOMPLETE** — no FAIL or BLOCKED exists, but one or more underlying checks returned NOT FOUND.

The deterministic precedence is:

- any FAIL → NOT READY
- else any BLOCKED → BLOCKED
- else any NOT FOUND → INCOMPLETE
- else all six PASS → READY

Native results are preserved: NOT FOUND is not rewritten as FAIL, BLOCKED is not rewritten, and unfavorable results are not hidden.

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

The Documentation Check outputs a block of the same shape. A passing result looks like:

```
Documentation Check: PASS
Documentation: README.md
Issues: 0
Result: PASS
Reason: The current canonical README's explicit mechanically verifiable claims are consistent with current repository evidence.
```

The Security / Configuration Check outputs a block of the same shape. A passing result looks like:

```
Security / Configuration Check: PASS
Issues: 0
Result: no confirmed high-confidence security/configuration findings
Reason: A meaningful tracked environment/configuration surface was inspected; its values were ordinary configuration, explicit placeholders, or variable references.
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

- The Documentation Check performs a read-only reconciliation between explicit current documentation claims and concrete current repository evidence. It verifies only explicit mechanically verifiable claims (paths, entry points, canonical commands, capability/status/roadmap consistency) and distinguishes examples and planned language from current-state claims. It does not execute test/lint/build commands, fetch external links, or modify documentation.

- The Security / Configuration Check inspects current non-ignored project state read-only, identifies a narrow high-confidence security/configuration surface, applies placeholder/example/fixture protections, never uses entropy-only guessing, refuses to follow security-relevant symlinks outside the repository, and reports sensitive findings using redacted metadata only.

- The Release Readiness Summary orchestrates the six existing checks in Skill-defined order. Each check retains its own discovery, current-state, execution, and safety rules. Native results are collected without rewriting, and the deterministic precedence produces the release-level result. No tag, release, publication, push, or deploy action is performed.

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

### Documentation Check

We performed a set of behavioral tests of the Documentation Check:

1. **Real Factory ShipCheck repository with stale committed README → FAIL.**
   - Current `SKILL.md` implemented five checks, including the Documentation Check.
   - `README.md` still said four checks were implemented.
   - The README Roadmap still listed documentation as planned/not implemented.
   - The README Status still said Documentation was not yet implemented.
   - ShipCheck treated root `README.md` as canonical, used current `SKILL.md` as concrete implementation evidence, and verified other explicit claims such as the Skill entry-point path.
   - It identified exactly three explicit capability/status/roadmap contradictions: the implemented capability count/current capabilities, the Roadmap still treating Documentation as planned, and the Status still treating Documentation as unimplemented.
   - It did not judge writing quality or subjective completeness, made no repository changes, executed no Test/Lint/Build commands, and fetched no external links.
   - Documentation Check → FAIL.
2. **Same real repository with README corrected in the CURRENT working tree but NOT committed → PASS.**
   - HEAD still contained the stale README.
   - The on-disk working-tree README had the correction.
   - ShipCheck read the current README directly from disk; HEAD was used only as supporting comparison evidence.
   - The current README documented five checks, Documentation was removed from the Roadmap, and Status described the Documentation Check as implemented with behavioral testing in progress.
   - No explicit mechanically verifiable contradictions remained; Issues: 0.
   - Documentation Check → PASS.
   - The pre-existing uncommitted README modification remained untouched.
   - This demonstrated the Documentation Check evaluates current on-disk documentation rather than silently substituting HEAD.
3. **Documentation demo with no canonical project documentation → NOT FOUND.**
   - The committed demo contained only `app.sh` and `.factory/skills/ship-check/SKILL.md`.
   - There was no README, no `docs/`, no CONTRIBUTING, no CHANGELOG, no LICENSE, and no explicit documentation-authority configuration.
   - ShipCheck searched for canonical/release-facing documentation and correctly treated `SKILL.md` as workflow/implementation evidence rather than automatically promoting it to project documentation.
   - It created no documentation and changed nothing.
   - Documentation Check → NOT FOUND.
   - Absence of canonical documentation is NOT FOUND, not FAIL.
4. **Same documentation demo with a temporary generated/non-authoritative README → BLOCKED.**
   - The temporary README explicitly stated it was generated output, that it was not the authoritative project documentation, and that the authoritative documentation lived outside the repository in a private documentation system unavailable from the local checkout.
   - ShipCheck found the README entry point, respected its explicit disclaimer of authority, and recognized that a different authoritative source was declared.
   - It confirmed that source was unavailable locally, did not treat the generated README as authoritative, did not invent or reconstruct the private source, did not fetch external/private documentation, and did not report a false contradiction.
   - It made no repository changes.
   - Documentation Check → BLOCKED.
   - This is not NOT FOUND because documentation plus an explicit authority declaration existed; not FAIL because no concrete contradiction with authoritative documentation was established; and not PASS because the authoritative source could not be inspected. Therefore BLOCKED.

These experiments demonstrate four distinct Documentation Check responsibilities: identifying documentation authority, reconciling explicit documentation claims with concrete repository evidence, evaluating current uncommitted documentation rather than only HEAD, and refusing to guess when documentation authority cannot be resolved.

### Security / Configuration Check

We performed a set of behavioral tests of the Security / Configuration Check:

1. **Real Factory ShipCheck repository with no meaningful v1 security/configuration surface → NOT FOUND.**
   - The current repository was inspected read-only.
   - No meaningful in-scope environment/configuration/credential/private-key surface was identified.
   - Ordinary source/documentation prose did not become findings.
   - Ignored local secret contents were not inspected.
   - Git history was not scanned.
   - No Test/Lint/Build command was executed.
   - No network request was made.
   - Security / Configuration Check → NOT FOUND.
   - "Nothing security-specific to inspect" is NOT FOUND, not PASS.
2. **Security demo with a tracked, non-ignored `.env` → PASS.**
   - The committed `.env` contained `APP_ENV=demo`, `API_TOKEN=CHANGEME`, and `DATABASE_PASSWORD=${DATABASE_PASSWORD}`.
   - `.env` created a meaningful v1 configuration surface.
   - `APP_ENV=demo` was ordinary configuration.
   - `API_TOKEN=CHANGEME` was an explicit placeholder.
   - `DATABASE_PASSWORD=${DATABASE_PASSWORD}` was a variable reference rather than a stored credential.
   - No supported high-confidence finding was present.
   - Security / Configuration Check → PASS.
   - This experiment proved the PASS versus NOT FOUND distinction.
3. **Same security demo with one temporary non-ignored synthetic private-key-shaped artifact → FAIL.**
   - The temporary file was named `synthetic-private-key.pem`.
   - It contained an unmistakable private-key boundary structure.
   - The content was deliberately synthetic/nonfunctional for the experiment.
   - ShipCheck established the finding using structural metadata; cryptographic key validation was not required.
   - Security / Configuration Check → FAIL.

   **Redaction proof:** during the actual Security Check run, the body was NOT printed, the complete file was NOT printed, no credential value was reproduced as evidence, and only safe path/type/structural metadata was reported. Example of the safe finding style (the synthetic body is not reproduced here):

   ```
   Finding 1:
   Type: private key
   Path: synthetic-private-key.pem
   Evidence: PEM private-key header detected; contents redacted.
   Issue: Private-key material is present in current non-ignored project content.
   ```

   The temporary artifact was subsequently deleted and the exact committed PASS baseline restored.
4. **Same security demo with a security-relevant external symlink → BLOCKED.**
   - `README.md` explicitly identified `release-secrets.env` as current release security configuration.
   - `release-secrets.env` was a symlink whose target resolved outside the repository.
   - The Security / Configuration Check rules forbid following external symlink targets.
   - ShipCheck used only safe symlink metadata.
   - It did NOT dereference `release-secrets.env`.
   - It did NOT inspect the external target contents.
   - It did NOT invent a FAIL finding based on the filename.
   - It could not safely establish PASS because the relevant security configuration could not be inspected.
   - The meaningful surface prevented NOT FOUND.
   - Therefore Security / Configuration Check → BLOCKED.
   - Cleanup removed both the repository symlink and the external synthetic temporary file, and returned the demo to the exact committed PASS baseline.

These experiments demonstrate four separate responsibilities: distinguishing absence of a meaningful security surface from a clean inspectable surface, recognizing high-confidence current security/configuration findings, avoiding false positives for placeholders and variable references, and enforcing safety boundaries and protecting sensitive material in ShipCheck's own output.

## Roadmap

The current ShipCheck readiness roadmap is implemented: six readiness checks plus the Release Readiness Summary orchestrator.

## Built With

This project is being built and tested with Factory Droid. The initial development used GLM-5.2 (Droid Core), but the Skill does not require that specific model.

## Status

ShipCheck is an early work-in-progress. The Git State Check is implemented and behaviorally tested. The Test Check is implemented and behaviorally tested across PASS, FAIL, NOT FOUND, and BLOCKED. The Lint Check is implemented and behaviorally tested across PASS, FAIL, NOT FOUND, and BLOCKED. The Build Check is implemented and behaviorally tested across PASS, FAIL, NOT FOUND, and BLOCKED. The Build Check isolation was behaviorally tested by allowing build artifacts to be generated in a disposable workspace while the original repository remained clean. The Build Check BLOCKED behavior was tested for both remote publication/upload side effects and unavailable required tooling. The Documentation Check is implemented and behaviorally tested across PASS, FAIL, NOT FOUND, and BLOCKED. The Documentation Check's current-state behavior was tested with a corrected but uncommitted README. The Documentation Check's BLOCKED behavior was tested with an explicit non-authoritative generated README whose declared authoritative source was unavailable. The Security / Configuration Check is implemented and behaviorally tested across PASS, FAIL, NOT FOUND, and BLOCKED. The Security / Configuration Check secret-output redaction was behaviorally verified. The Security / Configuration Check's external-symlink safety boundary was behaviorally verified. The Release Readiness Summary is implemented. Release Readiness Summary behavioral testing is in progress. Its NOT READY aggregation has been behaviorally exercised with Git State PASS, Documentation FAIL, four NOT FOUND results, all six checks still executed, native results preserved, counts summing to six, and final result NOT READY via FAIL precedence.
