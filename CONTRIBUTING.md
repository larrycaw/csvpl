# Contributing to CSVPL

Thanks for your interest in CSVPL. This guide covers the workflow, build, and
review expectations for contributions.

## Prerequisites

- JDK 11 (current baseline; moving to 17 as part of the platform refresh — see
  `SUPPORT.md`).
- Maven 3.6+.

## Build & test

```bash
# Run the full unit suite.
mvn test

# Run a single test.
mvn test -Dtest=ShellTest

# Build without tests.
mvn -DskipTests package
```

Release-only plumbing (GPG signing, sources/javadoc attach, Sonatype staging)
is gated behind the `release` profile and does not run during `mvn test`.
See PR 3 of the platform refresh plan for the profile layout.

## Workflow

1. Open an issue for anything larger than a typo fix, so scope can be agreed
   before code is written.
2. Branch from `master`. Use a descriptive branch name
   (`fix/csv-parser-empty-line`, `feat/streaming-pipeline`).
3. Keep PRs focused. One concern per PR — a dependency bump bundled with a
   refactor makes `git bisect` much less useful.
4. Every PR must:
   - pass CI (tests, lint, security scans),
   - include tests for new behavior or a note in the description if no test
     is practical,
   - update docs and ADRs when changing architecture.
5. Use a commit message that explains *why*. The first line is a short
   imperative summary ("Pin slf4j-log4j12 to 1.7.36"); the body explains
   motivation and trade-offs.

## Review expectations

- At least one code owner review is required (see `.github/CODEOWNERS`).
- Changes to public API (classes under `ai.preferred.csvpl.api` once PR 7
  of the refresh lands) require two reviewers and a note in the PR
  description about the compatibility policy in `COMPATIBILITY.md`.
- Reviewers should respond within two business days or reassign.

## Coding conventions

- Java 11 source level until the Java 17 bump lands (PR 4 of the refresh).
- Four-space indent, no tabs.
- Avoid `System.exit` in library code (PR 8 of the refresh will enforce this).
- Prefer SLF4J over `System.out` for diagnostics (PR 6 of the refresh).
- Public classes: add a short class-level Javadoc describing intent, not
  implementation.

## Reporting security issues

Do not open a public issue for security vulnerabilities. See `SECURITY.md`
(added in PR 12 of the refresh) for the private disclosure process.

## Releases

See the release checklist in `docs/releases.md` (added in PR 13 of the
refresh). Until then, releases follow the legacy OSSRH staging flow
triggered on tag push.
