# Validation — Plan 001: CI unit-test gate

Date: 2026-10-12
Repository: `/Users/ohmskiii/ghidra`
Baseline: clean `master` mirror before this change

## Finding and remediation

The existing push workflow `.github/workflows/build-ghidra.yml:29-30` only runs
`./gradlew buildGhidra --parallel`. `gradle/root/distribution.gradle:703-710`
shows `buildGhidra` builds the distribution; its dry-run task graph contained
zero test tasks. A separate workflow, `.github/workflows/test-ghidra.yml`, now
runs the existing aggregate `unitTestReport` task on pull requests and pushes
to `master`, with the repository JDK setup, dependency fetch, and Xvfb required
by `DevGuide.md:171-176`. The distribution workflow is unchanged.

The audit found the README's JDK 21 install vs JDK 25 build requirements were
not contradictory: Install applies to official pre-built releases; Build
applies to this development source tree. This false-positive candidate was
rejected.

## Verification evidence

| Check | Result |
|---|---|
| `./gradlew buildGhidra --dry-run --console=plain` | Exit 0; zero test tasks in graph |
| `./gradlew unitTestReport --dry-run --console=plain` | Exit 0; 1,054 tasks, including module tests and `unitTestReport` |
| `./gradlew :Generic:test --no-daemon --console=plain` | **BUILD SUCCESSFUL** in 2m28s; 116 actionable tasks (81 executed, 35 up-to-date) |
| Ruby YAML parse of `test-ghidra.yml` | Parsed successfully; `actionlint` unavailable, so GitHub Actions schema was not independently linted |
| `git diff --check` | Passed; no whitespace errors |
| Existing `build-ghidra.yml` diff | Empty; unchanged |

## Aggregate unit suite: partial, failures observed

Attempted:

```sh
./gradlew unitTestReport --parallel --no-daemon --console=plain
```

Output was captured at `/tmp/ghidra-unitTestReport-20261012020518.log` (now
archived, gzip-compressed, at
`plans/archive/2026-10-12/ghidra-unitTestReport-20261012020518.log.gz`). It
ended while test tasks were still printing output; **there is no final
`BUILD SUCCESSFUL` or `BUILD FAILED` summary**. Do not treat this as a completed
suite. The available fresh XML reports (timestamps 02:05:35–02:08:21 PDT) cover
308 report files and 3,417 tests: 3,334 passed, 42 failures, 0 errors, and 41
skipped. These are partial task results, not aggregate completion totals.

Observed failure groups include:

- 7zip CAB tests unable to initialize native libraries for this host's
  `aarch64` macOS platform (the logged library list has Linux-amd64,
  Mac-x86_64, and Windows variants, but no Mac-aarch64).
- macOS PTY tests timing out.
- GUI/action tests failing with invalid action context or host-dependent
  display-string expectation.
- BSim setup tests failing with null fixture objects, and CodeCompare asserts.

Because the local host is macOS/aarch64 while CI uses Ubuntu, local `Xvfb` is
unavailable, and the command did not complete, this run cannot establish which
failures reproduce on the target CI runner or whether they are pre-existing.
The new GitHub Actions job has not run: this branch/workflow has not been
pushed. The full suite must be re-run to completion on Ubuntu/Xvfb before
marking the plan fully verified.

The eight literal untracked log filenames (`${sys:logFilename}` and
`${sys:scriptLogFilename}`) created by tests were removed. Normal ignored Gradle
build/test outputs are preserved.

## Audit scope and limitations

The four delegated read-only audit sessions timed out without reports. This
validation therefore covers the directly verified CI/test finding and the
limited docs sanity check above; it is not a claim that the security,
correctness, dependency, performance, and whole-repo documentation categories
were exhaustively audited. No security finding was validated or rejected by
those timed-out passes.

## Current acceptance state

- Implementation and local targeted checks: complete.
- Workflow syntax: basic YAML parse passed; GitHub Actions-specific lint not
  available locally.
- Full aggregate test suite: incomplete; failures observed, requires rerun and
  triage on the Ubuntu runner.
- GitHub CI: pending until branch is pushed.
- Plan 001 status: **IMPLEMENTED; CI pending**.
