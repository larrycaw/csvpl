# CSVPL Option C (Platform Refresh) — PR Sequence

This document breaks the Option C modernization plan into a meaningful sequence
of pull requests. Each PR is scoped to be independently reviewable and
mergeable, with explicit dependencies, acceptance criteria, and rollback notes.

The sequence is grouped into four waves. Waves are ordered by dependency:
earlier waves unblock later ones. Within a wave, PRs that do not depend on each
other may be opened and reviewed in parallel (noted where relevant).

## Ordering principles

- **Ship the safety net first.** CI, governance, and pinning land before any
  code refactor so regressions surface immediately.
- **One concern per PR.** Avoid mixing a Java version bump with a dependency
  bump with a refactor — `git bisect` stays useful.
- **Behavior-preserving before behavior-changing.** Architectural redesign PRs
  are preceded by tests and benchmarks that pin current behavior.
- **Docs close each wave.** Every wave ends with the ADR(s) and user-facing
  docs it produced; docs are not deferred to the end of the program.

## Summary

| # | PR | Wave | Depends on | Risk |
|---|----|------|------------|------|
| 1 | Repo governance & ADR bootstrap | 0 | — | Low |
| 2 | CI migration to GitHub Actions (parity) | 0 | — | Low |
| 3 | Build hygiene & deterministic dependencies | 0 | 2 | Low |
| 4 | Java 17 LTS baseline | 1 | 3 | Medium |
| 5 | Dependency upgrades (Weka, Commons, JUnit 5, Guava) | 1 | 4 | Medium |
| 6 | Logging backend modernization (SLF4J + Log4j2) | 1 | 5 | Low |
| 7 | Explicit extension registry & package boundaries | 2 | 4, 5 | Medium |
| 8 | CLI/API normalization & error model | 2 | 7 | Medium |
| 9 | Streaming CSV pipeline redesign | 2 | 8 | High |
| 10 | Validation suite (unit + integration + golden) | 3 | 7, 8 | Low |
| 11 | Performance benchmarks & regression gate | 3 | 9, 10 | Medium |
| 12 | Static analysis & security scans as required checks | 3 | 2, 5 | Low |
| 13 | Release automation (Sonatype Portal, signing, changelog) | 4 | 3, 12 | Medium |
| 14 | Documentation set (quickstart, architecture, migration) | 4 | 7–9, 13 | Low |

---

## Wave 0 — Foundations (enable safe change)

### PR 1 — Repo governance & ADR bootstrap
- **Scope:** Add `CODEOWNERS`, `CONTRIBUTING.md`, issue & PR templates, a
  `SUPPORT.md` stating the Java/LTS support policy, a `COMPATIBILITY.md`
  draft defining public-API and deprecation policy, and a `docs/adr/`
  directory seeded with ADR-0000 ("record architectural decisions").
- **Depends on:** —
- **Acceptance:** Files render on GitHub; no code changes.
- **Rollback:** Revert single commit.

### PR 2 — CI migration to GitHub Actions (parity)
- **Scope:** Replace `.travis.yml` with `.github/workflows/ci.yml`. Run the
  existing `mvn clean test` on push/PR. Add a matrix for JDK 11 (current)
  only; do **not** bump Java yet. Wire Maven cache, JUnit report upload, and
  Coveralls via GHA. Keep Travis file for one release cycle behind a comment
  flag, then remove in a follow-up.
- **Depends on:** —
- **Acceptance:** CI green on PR branch; coverage reports visible; README
  badge updated.
- **Rollback:** Delete workflow; Travis file remains functional until PR 3
  removes `versions:resolve-ranges` workaround.

### PR 3 — Build hygiene & deterministic dependencies
- **Scope:** Pin every plugin version in `pom.xml`; remove all Maven
  version ranges (the `before_install: mvn versions:resolve-ranges` smell
  in `.travis.yml` goes away). Split Maven profiles: `dev` (default, no GPG,
  no javadoc), `ci`, `release` (GPG + staging). Drop `nexus-staging` plugin
  config from the default lifecycle so `mvn test` does not touch release
  plumbing. Delete `.travis.yml`.
- **Depends on:** PR 2
- **Acceptance:** `mvn -o test` succeeds with a warm cache; `mvn -P release`
  runs the staging flow; `mvn dependency:tree` stable across two runs.
- **Rollback:** Revert; nothing downstream consumes the new profiles yet.

## Wave 1 — Platform baseline

### PR 4 — Java 17 LTS baseline
- **Scope:** Bump `maven.compiler.source/target` to 17. Remove deprecated
  `Class#newInstance()` usages (e.g., in `Shell`/`Command` dispatch) in favor
  of `getDeclaredConstructor().newInstance()`. Replace obviously dead legacy
  idioms only where the compiler forces the change. Update CI matrix to
  `[17, 21]`.
- **Depends on:** PR 3 (needs deterministic build + new profiles)
- **Acceptance:** Green on JDK 17 and 21; no behavior change in `ShellTest`.
- **Rollback:** Single revert. No API shape change yet.
- **Out of scope:** Package moves, logging changes, streaming — those follow.

### PR 5 — Dependency upgrades
- **Scope:** Upgrade Weka, Commons-* , Guava, and test deps to current
  supported majors. Migrate JUnit 4 → JUnit 5 (Jupiter) in one sweep so
  test-writer ergonomics are fixed before Wave 3 expands the suite. Address
  transitive conflicts via `<dependencyManagement>`.
- **Depends on:** PR 4
- **Acceptance:** `ShellTest` passes on JUnit 5; no runtime regressions on
  the sample scripts in `data/`.
- **Rollback:** Revert, but coordinate with PR 6 which depends on the new
  logging API surface.

### PR 6 — Logging backend modernization
- **Scope:** Introduce SLF4J as the API across the codebase; Log4j2 as the
  runtime backend. Remove any `System.out.println`/`printStackTrace` error
  paths in favor of structured logger calls with MDC-style context (command
  name, script line). Ship a default `log4j2.xml` under `src/main/resources`
  with a sane console layout.
- **Depends on:** PR 5
- **Acceptance:** No logger-framework classes imported outside a single
  bootstrap class; logs include command + line context on failure.
- **Rollback:** Revert single commit; no API change.

## Wave 2 — Architecture boundaries (the actual "refresh")

### PR 7 — Explicit extension registry & package boundaries
- **Scope:** Split `ai.preferred.regression` into clearer boundaries:
  - `ai.preferred.csvpl.api` (stable public types: `Command`, `ProcessingElement`)
  - `ai.preferred.csvpl.core` (shell, dispatch, script parser)
  - `ai.preferred.csvpl.pe` (built-in processing elements)
  - `ai.preferred.csvpl.regression` (Weka-backed commands)

  Replace reflection-based command/PE lookup in `Shell` with an explicit
  `CommandRegistry` / `ProcessingElementRegistry` populated via
  `ServiceLoader`. Keep the old FQCNs as deprecated aliases that delegate,
  so external scripts continue to resolve. Land ADR-0001 ("extension
  registry").
- **Depends on:** PR 4, PR 5
- **Acceptance:** All existing scripts in `data/` run unchanged; a new PE
  can be registered by dropping a `META-INF/services` file; deprecation
  warnings logged for legacy FQCN resolution.
- **Rollback:** Aliases make the diff reversible per-package; revert in
  reverse order of the commits within the PR.

### PR 8 — CLI/API normalization & error model
- **Scope:** Remove `System.exit` from `Shell`/`Command` bodies. Introduce a
  `CsvplException` hierarchy (`ScriptParseException`, `CommandFailure`,
  `IoFailure`) and a single top-level exit-code mapper in `Main`. Normalize
  command argument validation (required/optional, typed). Improve help text
  emitted per command. Land ADR-0002 ("error model & exit codes").
- **Depends on:** PR 7
- **Acceptance:** `Shell` is embeddable from a test without the JVM exiting;
  malformed script lines surface a diagnostic pointing at file:line; exit
  codes documented.
- **Rollback:** Revertable; callers inside this repo are limited to `Main`
  and tests.

### PR 9 — Streaming CSV pipeline redesign
- **Scope:** Replace whole-file `List<String[]>` in-memory processing with a
  streaming `CsvReader`/`CsvWriter` abstraction (chunked, backed by a
  well-maintained CSV lib — e.g. `univocity-parsers` or `commons-csv`).
  Define charset (UTF-8 default, overridable), dialect, quoting, and
  malformed-row policy in one place. Processing elements that genuinely
  need the full dataset (sort, shuffle, partition) declare that need
  explicitly and spill to disk above a configurable threshold. Land
  ADR-0003 ("streaming pipeline & large-file policy").
- **Depends on:** PR 8
- **Acceptance:** Existing golden outputs (added in PR 10) match
  byte-for-byte for fixture datasets; memory profile on a 1 GB CSV stays
  bounded for non-materializing PEs.
- **Rollback:** Highest-risk PR in the plan. Keep the legacy in-memory
  path behind a feature flag (`csvpl.pipeline=legacy|streaming`,
  default `legacy` in this PR) and flip the default in a follow-up after a
  release cycle.

## Wave 3 — Validation & performance

### PR 10 — Validation suite
- **Scope:** Expand tests from the single `ShellTest` smoke to:
  - unit tests per PE (parameterized for edge cases: empty input, unicode,
    quoted commas, mixed line endings),
  - integration tests per command,
  - golden-output regression tests over scripts in `data/` (stored under
    `src/test/resources/golden/`).

  Adds `@Nested` JUnit 5 structure, a `CsvplTestHarness` helper, and
  deterministic temp-dir handling.
- **Depends on:** PR 7, PR 8 (needs embeddable Shell and registry)
- **Acceptance:** Line coverage ≥ 70% on `core` and `pe` packages; golden
  diffs required for any PE change going forward.
- **Rollback:** Tests are additive; revertable.

### PR 11 — Performance benchmarks & regression gate
- **Scope:** Add a JMH module under `benchmarks/` covering read, transform
  (project/select/encode), train-linear, apply-linear. Capture a baseline
  in `benchmarks/baselines/`. Add an optional CI job that runs a quick
  benchmark and fails PRs that regress a critical path by >15% on a small
  fixture (advisory in this PR, required once stable).
- **Depends on:** PR 9, PR 10
- **Acceptance:** Baseline checked in; bench job runs in <5 min on CI;
  advisory-only status initially.
- **Rollback:** Gate is advisory; disabling it is a single workflow edit.

### PR 12 — Static analysis & security scans
- **Scope:** Add SpotBugs + ErrorProne to the Maven build under a `lint`
  profile, wired into CI as a required check. Add CodeQL workflow, Dependabot
  (weekly, grouped), and OSSF secret scanning. Fail CI on new High/Critical
  advisories; existing findings captured in a baseline file.
- **Depends on:** PR 2, PR 5
- **Acceptance:** Three required checks on PRs: build, CodeQL, lint. Baseline
  file documents pre-existing findings with owners/issues.
- **Rollback:** Each scanner is a separate commit; revert individually.

## Wave 4 — Release & docs

### PR 13 — Release automation
- **Scope:** Migrate from legacy OSSRH `nexus-staging` to the new Sonatype
  Central Portal publishing flow. Adopt `central-publishing-maven-plugin`.
  Move signing keys from the Travis-encrypted blob to GitHub Actions
  Environments + encrypted secrets. Add release-drafter (auto-changelog from
  Conventional Commits) and a `release.yml` workflow triggered on tag push.
  Add a release checklist template.
- **Depends on:** PR 3 (profiles), PR 12 (signed artifacts pass scans)
- **Acceptance:** Dry-run release to a staging repository succeeds from GHA;
  changelog generated from merged PRs; encrypted `key.gpg.enc` removed from
  the repo.
- **Rollback:** Keep the old profile behind `-P release-legacy` for one
  release; remove it in the next cycle.

### PR 14 — Documentation set
- **Scope:**
  - Refresh `README.md` (badges, current Java version, quickstart for CLI
    and library usage, link to migration guide).
  - `docs/architecture.md` with the post-refresh component diagram.
  - `docs/extensions.md` explaining how to author a new `Command` or
    `ProcessingElement` against the registry.
  - `docs/migration-1.x-to-2.x.md` cataloguing removed APIs, new error
    model, streaming pipeline flag, and deprecation timeline.
  - ADR index at `docs/adr/README.md`.
- **Depends on:** PR 7–9 (so the docs describe the final shape), PR 13
- **Acceptance:** Docs build via `mvn site` (or the chosen docs generator);
  internal links pass a link-checker CI step.
- **Rollback:** Pure docs; revertable.

---

## Release checkpoints

- **After Wave 0:** patch release (e.g., `1.0.1`) — safer build, modern CI.
- **After Wave 1:** minor release (`1.1.0`) — new Java floor announced via
  `SUPPORT.md`.
- **After Wave 2:** major release candidate (`2.0.0-rc1`) — new architecture,
  streaming flag defaulting to `legacy`.
- **After Wave 3:** `2.0.0-rc2` — streaming flag flipped to default on.
- **After Wave 4:** `2.0.0` GA — release automation + full docs.

## What is explicitly *not* a separate PR

- A big-bang rewrite. Each PR keeps the product shippable.
- Package renames bundled with behavior changes. PR 7 is rename+registry
  only; PR 8 changes behavior on top of stable names.
- Dependency bumps bundled with refactors. PR 5 is dependency-only so a
  Weka regression is not entangled with logging or pipeline changes.
