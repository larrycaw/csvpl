# CSVPL Support Policy

## Getting help

- **Questions & discussion:** open a GitHub Discussion.
- **Bug reports:** open a GitHub Issue using the bug-report template.
- **Feature requests:** open a GitHub Issue using the feature-request template.
- **Security issues:** see `SECURITY.md` (added in PR 12 of the platform
  refresh) — do not use public issues for vulnerabilities.

## Java version policy

CSVPL follows the [JDK LTS cadence](https://www.oracle.com/java/technologies/java-se-support-roadmap.html).

| CSVPL version | Minimum JDK | Tested JDKs | Status |
|---|---|---|---|
| 1.x | 8 | 11 | Maintenance — critical fixes only. |
| 2.x *(planned)* | 17 | 17, 21 | Active development (platform refresh). |
| 3.x *(future)* | Next LTS | Next LTS + current | Not yet scheduled. |

- New minor releases of an active major line support the same minimum JDK
  as the `.0` release.
- A new major line may raise the minimum JDK. Such a bump is announced at
  least one minor release in advance.
- We test on every LTS JDK between the minimum and the latest GA, plus
  the latest non-LTS if free CI capacity allows.

## Support windows

- **Active:** the most recent major line receives feature and bug work.
- **Maintenance:** the previous major line receives critical and security
  fixes for 12 months after the next major's `.0` GA.
- **End of life:** older lines receive no further releases. Users should
  plan upgrades before EOL.

## Dependency policy

- Critical dependencies (Weka, Commons-CSV, SLF4J, logging backend) are
  pinned to a specific version. See `COMPATIBILITY.md` for how upgrades
  are classified.
- We aim to track the latest patch of each pinned dependency within one
  release cycle of its publication, except when doing so would break the
  compatibility promise.

## Reporting a regression

If an upgrade from N to N+1 within the same major line breaks your usage,
file an issue tagged `regression`. Regressions within a major line are
treated as release blockers.
