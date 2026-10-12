# Plan 001: Run unit tests for pull requests and master pushes

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in the STOP conditions occurs, stop and report.
>
> **Drift check**: `git diff --stat HEAD -- .github/workflows/`
> This plan was written against the clean `master` tree at the audit baseline.

## Status

- **Priority**: P1
- **Effort**: M
- **Risk**: LOW
- **Depends on**: none
- **Category**: tests
- **Planned at**: clean `master`, 2026-10-12

## Why this matters

The repository contains 1,145 Java test files, but the only CI workflow runs
`./gradlew buildGhidra --parallel`. The verified `buildGhidra --dry-run` task
graph contains zero test tasks, so a build can pass while a regression in the
test-covered code goes undetected. Pull requests also have no workflow trigger,
so contributors receive no repository-native verification before merge.

## Current state

- `.github/workflows/build-ghidra.yml:3` uses `on: [push]`.
- `.github/workflows/build-ghidra.yml:29-30` runs only
  `./gradlew buildGhidra --parallel`.
- `gradle/root/test.gradle:391-397` defines `unitTestReport` as the aggregate
  unit-test task.
- `DevGuide.md:153-167` documents `unitTestReport`, `integrationTest`, and
  `combinedTestReport`.
- `DevGuide.md:171-176` requires Xvfb for headless/CI test execution.
- Verified commands:
  - `./gradlew buildGhidra --dry-run` → no test tasks in the graph.
  - `./gradlew unitTestReport --dry-run` → 1,054 tasks.
  - `./gradlew combinedTestReport --dry-run` → 1,426 tasks.

## Scope

**In scope:**

- `.github/workflows/test-ghidra.yml` (create)

**Out of scope:**

- Existing `.github/workflows/build-ghidra.yml` behavior.
- Production Java code.
- Integration-test scheduling.
- New dependencies, caching, or matrix expansion.

## Implementation

Create a separate `Test Ghidra` workflow with:

1. `pull_request` trigger for all branches.
2. `push` trigger limited to `master`, so the merged result is re-verified.
3. Ubuntu runner, matching the current build workflow's supported CI platform.
4. The same JDK discovery and Temurin setup used by
   `.github/workflows/build-ghidra.yml:17-24`.
5. The same dependency fetch command used by
   `.github/workflows/build-ghidra.yml:26-27`.
6. Xvfb startup with `DISPLAY=:99`, as required by `DevGuide.md:171-176`.
7. `./gradlew unitTestReport --parallel` as the test command.

Keep the build workflow unchanged. This prevents a passing distribution build
from masking a failed test job and avoids changing release-build behavior.

## Verification

Run locally:

```bash
./gradlew unitTestReport --dry-run --console=plain
```

Expected: exit 0 and the task graph includes module `test` tasks.

Validate workflow syntax by inspecting the YAML and running the existing local
Gradle dry run. GitHub Actions is the authoritative execution environment for
the new workflow; a completed CI job must show `Test Ghidra / unit-tests` green.

## Done criteria

- [x] `.github/workflows/test-ghidra.yml` exists and parses as YAML.
- [x] Workflow runs on `pull_request` and on pushes to `master` only.
- [x] Workflow sets JDK from `Ghidra/application.properties`.
- [x] Workflow fetches dependencies, starts Xvfb, and runs
      `./gradlew unitTestReport --parallel`.
- [x] Existing `build-ghidra.yml` is unchanged.
- [x] `./gradlew unitTestReport --dry-run --console=plain` exits 0 and includes
      test tasks.
- [x] No production source files change.
- [ ] GitHub Actions completes `unit-tests` successfully on Ubuntu/Xvfb.

Verified locally 2026-10-12: Ruby YAML parse succeeded, `git diff --check`
passed, `unitTestReport --dry-run` exited 0, and `:Generic:test` passed in
2m28s (116 actionable tasks, 81 executed, 35 up-to-date). The full aggregate
suite was interrupted before Gradle emitted a final summary; see the session
validation report. GitHub Actions execution remains pending because this
branch/workflow has not been pushed.

## STOP conditions

- Stop if the current JDK setup or dependency-fetch command differs from the
  excerpts above.
- Stop if `unitTestReport` no longer creates/runs module test tasks.
- Stop if workflow validation requires adding a dependency or changing
  production code.
- Stop if repository policy requires a different workflow name, runner, or
  trigger convention.
