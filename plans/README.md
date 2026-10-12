# Improvement Plans

Generated from the verified audit of this Ghidra mirror on 2026-10-12.

| Plan | Title | Priority | Effort | Depends on | Status |
|---|---|---:|---:|---|---|
| 001 | Run unit tests for pull requests and master pushes | P1 | M | none | IMPLEMENTED; CI pending; local suite partial (42 failures observed) |

## Findings considered and rejected

- README JDK 21 vs JDK 25: by design. The Install section documents official
  pre-built releases; the Build section documents this source tree.
- Full combined test coverage in CI: deferred. `combinedTestReport` expands to
  1,426 dry-run tasks and includes environment-sensitive integration tests. The
  first safe gate is unit coverage; integration coverage can follow as a
  separately measured change.
