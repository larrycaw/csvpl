# CSVPL Compatibility & Deprecation Policy

This document defines what "compatible" means for CSVPL releases and how we
phase out APIs. The policy takes effect with `2.0.0`; `1.x` releases predate
it.

## Public API surface

From `2.0.0` onwards the **public API** is exactly the set of types and
members under:

- `ai.preferred.csvpl.api.**`
- Scripting surface: command names, processing-element names, their
  documented arguments, and documented option keys.
- CLI flags documented in `docs/cli.md`.
- Exit codes documented in `docs/cli.md`.

Everything else — including every type under `ai.preferred.csvpl.core`,
`ai.preferred.csvpl.pe`, and `ai.preferred.csvpl.regression` — is
**internal** and may change without notice in any release.

Packages not on this list are internal, even if they are reachable from a
user classpath.

## Semantic versioning rules

CSVPL follows [SemVer](https://semver.org/) with the public API defined
above:

- **Patch** (`x.y.Z`): bug fixes; no API or behavior change. No new
  scripting commands, no new options, no new exit codes.
- **Minor** (`x.Y.0`): backwards-compatible additions. New commands, new
  optional arguments, new option keys, new exit codes are allowed.
- **Major** (`X.0.0`): may remove or rename public API, change exit codes,
  or change command behavior. Requires a migration guide in `docs/`.

## Deprecation process

To remove a public API member:

1. **Mark deprecated** in the first minor release after the decision. Add
   `@Deprecated(since = "X.Y", forRemoval = true)` on Java members and a
   deprecation note in the help text for scripting surface. A deprecation
   must include a replacement or a rationale.
2. **Document** the deprecation in the release notes and in a new
   `docs/migration-X-to-Y.md` entry.
3. **Emit a runtime warning** through the logging framework on first use
   per JVM, with `WARN` level and a stable message containing the
   replacement.
4. **Remove** no sooner than the next major release, and no sooner than
   **two** minor releases after the deprecation was announced, whichever
   is later.

## Backwards-compatible aliases

When a public class or command is renamed, the old name remains as an
alias for the deprecation window. Aliases:

- forward to the new implementation,
- are annotated `@Deprecated`,
- log a one-time warning on use.

## Dependency classification

| Class | Examples | Upgrade rule |
|---|---|---|
| Public transitive | SLF4J API | Minor bumps allowed in minors; major bumps only in majors. |
| Private runtime | Weka, Commons-CSV | Any bump allowed in any release if behavior is preserved. |
| Test-only | JUnit, AssertJ | Any bump allowed in any release. |

## Script compatibility

- A `.csvpl` script that worked against `X.Y.0` must work against any
  `X.Y.Z` and any `X.Y+n.Z` in the same major.
- A major release may break scripts, but:
  - Every breaking script change is listed in the migration guide.
  - Where possible, the previous behavior is available behind a flag for
    at least one minor release of the new major.

## Exceptions

The policy may be waived only for:

- **Security fixes** that cannot be backported compatibly.
- **Undocumented, accidental** public exposure of internal types, where
  the originally-intended scope was internal.

Every waiver is called out explicitly in the release notes.
