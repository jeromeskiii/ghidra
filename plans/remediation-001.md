# Remediation Report — CI test coverage and pull-request validation

Date: 2026-10-12
Repository: `/Users/ohmskiii/ghidra`
Severity: Medium

## Scope

This report covers two related CI reliability findings in the pre-existing
GitHub Actions configuration. It does not claim an exhaustive audit of this
3-million-line codebase; the broader delegated audit passes timed out without
reports.

## MED-1: Distribution CI did not execute unit tests

### Evidence

- `.github/workflows/build-ghidra.yml:29-30` runs only
  `./gradlew buildGhidra --parallel`.
- `gradle/root/distribution.gradle:703-710` defines `buildGhidra` as a
  distribution-zip task.
- `./gradlew buildGhidra --dry-run --console=plain` exited 0 with zero `test`
  tasks in its graph.
- The repository contains 1,145 Java test files.
- `gradle/root/test.gradle:391-397` defines `unitTestReport` as the aggregate
  unit-test task.

### Impact

A change could compile and package successfully while breaking code with unit
test coverage. The existing CI signal therefore established distribution
integrity, not behavioral regression safety.

### Root cause

The sole workflow used the distribution task as its entire verification gate.
Ghidra intentionally separates distribution assembly from test orchestration,
so tests are not transitively executed by `buildGhidra`.

### Remediation

Add a separate `Test Ghidra` workflow that runs the repository-native
`unitTestReport` task. Keep the distribution workflow unchanged so release
packaging and test signals remain independently diagnosable.

### Acceptance criteria

- A unit-test job executes on pull requests and `master` pushes.
- It reads the required JDK from `Ghidra/application.properties`.
- It fetches dependencies, starts Xvfb, and runs
  `./gradlew unitTestReport --parallel`.
- Existing distribution CI remains unchanged.

## MED-2: Pull requests received no GitHub Actions validation

### Evidence

- `.github/workflows/build-ghidra.yml:3` used `on: [push]` only.
- `.github/workflows/` contained no test workflow before this change.
- The repository’s only CI job ran after pushes rather than as PR feedback.

### Impact

Reviewers had no repository-native build or test result attached to a pull
request. Regressions could be merged before the build workflow received a
push, increasing review uncertainty and time-to-detection.

### Root cause

CI was configured as a post-push distribution build, not as a pull-request
validation gate.

### Remediation

The new workflow includes a `pull_request` trigger and retains a `push` trigger
limited to `master`, verifying both proposed changes and the merged result.

### Acceptance criteria

- Opening or updating any pull request schedules `unit-tests`.
- Pushing to `master` schedules the same job.
- Unrelated branch pushes do not schedule the new job.

## Validation and remaining risks

See [validation-001.md](validation-001.md) for commands and observed output.
Local targeted testing passed. The aggregate unit run did not complete and
observed host-dependent failures on macOS/aarch64; GitHub Actions execution on
Ubuntu/Xvfb remains required before this remediation can be considered fully
verified.
