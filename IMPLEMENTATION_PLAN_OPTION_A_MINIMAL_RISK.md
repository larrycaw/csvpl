# CSVPL Modernization Plan — Option A (Minimal Risk)

## Goal
Improve reliability, security posture, and maintainability while preserving current architecture and behavior.

## Differentiator
This option explicitly avoids major architectural/API changes and prioritizes safe incremental hardening.

## Scope
- Maven/plugin/dependency refresh within backward-compatible limits.
- Deterministic build setup and clearer profiles.
- Logging backend modernization with minimal runtime behavior change.
- Core test expansion around existing shell and processing elements.
- CI migration from Travis to GitHub Actions with equivalent checks.
- Documentation refresh (README, contributor basics, release runbook).

## Non-goals
- No broad package restructuring.
- No major command model redesign.
- No disruptive API surface changes.

## Suggested phases
1. Build + dependency hygiene baseline.
2. CI migration and quality gates.
3. Targeted reliability fixes and test coverage increase.
4. Documentation and release workflow cleanup.
