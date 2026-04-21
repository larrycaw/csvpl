# CSVPL Modernization Plan — Option B (Balanced Modernization)

## Goal
Modernize platform fundamentals and selectively refactor internals for cleaner extension and better testability.

## Differentiator
Balances pragmatic modernization with moderate internal refactoring, while keeping migration burden manageable.

## Scope
- Move to modern Java LTS target and update outdated APIs.
- Standardize dependency versions and remove risky version ranges.
- Refactor command execution/error handling to reduce `System.exit` coupling.
- Improve shell/processing element dispatch clarity and validation.
- Expand automated tests (unit + integration + data edge cases).
- Adopt GitHub Actions matrix builds and security/quality scans.
- Improve project documentation and public/internal API boundaries.

## Non-goals
- No full rewrite.
- No high-disruption multi-module split unless required by clear bottlenecks.

## Suggested phases
1. Platform baseline (Java/build/dependencies/CI).
2. Refactor command and shell reliability surfaces.
3. Test strategy expansion and quality gates.
4. Documentation + release process modernization.
