# CSVPL Modernization & Optimization Master Plan

This document inventories all major modernization/improvement opportunities and offers multiple implementation tracks so maintainers can choose based on risk and effort.

## Option Summary

- **Option A (Minimal Risk):** Stabilize and secure the existing design with low disruption.
- **Option B (Balanced Modernization):** Modern Java + modern CI + better quality gates with moderate refactoring.
- **Option C (Platform Refresh):** Broader redesign for long-term maintainability, API clarity, and performance.

## What makes the options different

| Dimension | Option A | Option B | Option C |
|---|---|---|---|
| Change scope | Build/test/dependency hardening | Hardening + selective refactor | Full architectural modernization |
| API surface impact | None/minimal | Minimal to moderate | Moderate to high (compat layer recommended) |
| Delivery style | Short incremental PRs | Phased releases | Multi-milestone program |
| Risk level | Low | Medium | High |
| Long-term payoff | Medium | High | Very high |

## Complete Modernization Opportunity Inventory

### 1) Runtime and language baseline
- Upgrade Java baseline from 8-era setup to modern LTS (17 or 21).
- Remove deprecated reflection patterns (`Class#newInstance`) and legacy constructs.
- Replace outdated idioms with safer, clearer language features where appropriate.

### 2) Build system and Maven lifecycle
- Update legacy plugins and lock versions consistently.
- Separate developer workflow from release workflow (avoid release-only concerns during normal `test`).
- Improve reproducibility (plugin/dependency pinning, deterministic outputs).
- Rationalize Maven profiles for local, CI, and release modes.

### 3) Dependency hygiene and security
- Upgrade old libraries (e.g., Guava/Commons/JUnit/logging stack) to supported versions.
- Eliminate version ranges for critical dependencies for deterministic builds.
- Introduce regular dependency audit and vulnerability scanning policy.
- Reassess transitive dependencies brought by Weka and visualization stack.

### 4) Logging and observability
- Migrate from old Log4j binding approach to a modern, maintained backend.
- Standardize logging style and error context quality.
- Ensure command failures emit actionable, structured diagnostics.

### 5) Error handling and CLI behavior
- Remove direct `System.exit` from core logic to improve testability/composability.
- Introduce consistent exception mapping and user-facing error messages.
- Harden command parsing/execution against malformed script lines and missing args.

### 6) File I/O and data processing robustness
- Improve path handling and temp/data directory safety.
- Harden CSV read/write behavior (dialect, escaping, malformed rows, charset guarantees).
- Define memory/performance constraints for large CSV workflows.

### 7) Test strategy and quality gates
- Expand from smoke tests to unit + integration coverage for commands and PEs.
- Add parameterized tests for data edge cases and parser behavior.
- Add regression fixtures and deterministic golden-output tests.
- Introduce mutation/static checks as quality gates when practical.

### 8) CI/CD and automation
- Replace legacy Travis pipeline with GitHub Actions.
- Add matrix testing across Java versions.
- Add cache strategy, test reports, and artifact retention.
- Add branch protection checks and required status gates.

### 9) Release engineering and publishing
- Modernize OSSRH/Maven Central publish flow (new Sonatype token model and release process).
- Add signed release verification and explicit release checklist automation.
- Generate changelog/release notes automatically from commits/PRs.

### 10) Project structure and architecture
- Clarify boundaries between CLI shell, processing elements, I/O, and model logic.
- Reduce reflection-heavy dispatch where explicit registries improve clarity.
- Consolidate duplicated command patterns and improve extension points.

### 11) API/UX modernization
- Define stable public API vs internal classes.
- Improve command naming consistency and discoverability.
- Improve help text and examples for each command/processing element.

### 12) Documentation and onboarding
- Refresh README badges/links/workflows and version guidance.
- Add quickstart for both library usage and CLI scenarios.
- Add contributor guide (build, test, release, coding conventions).
- Add architecture and extension documentation for new contributors.

### 13) Data assets and examples
- Formalize sample datasets and expected outputs.
- Separate tutorial/exercise content from production library path.
- Add scripted reproducible demos for core workflows.

### 14) Performance optimization track
- Benchmark critical operations (read, transform, train, apply).
- Identify avoidable copying/boxing/allocation hotspots.
- Evaluate stream/chunk processing for large files.
- Add performance regression checks for critical paths.

### 15) Governance and maintainability
- Add CODEOWNERS, issue/PR templates, and review expectations.
- Define compatibility/deprecation policy.
- Define support policy for Java/LTS versions.

## Recommended decision flow

1. Choose target risk level (A/B/C).
2. Confirm supported Java version policy.
3. Approve dependency/security baseline.
4. Approve CI migration approach.
5. Execute in small, reviewable phases with release checkpoints.
