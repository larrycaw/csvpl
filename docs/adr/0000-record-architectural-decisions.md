# ADR-0000: Record architectural decisions

- **Status:** Accepted
- **Date:** 2026-04-21
- **Deciders:** CSVPL maintainers

## Context

CSVPL is entering a platform refresh (Option C of the modernization plan)
that will change package boundaries, the extension model, the error model,
and the data-processing pipeline. Decisions of that scope need a
durable record — not a Slack thread, not a closed PR — so future
contributors understand *why* the code looks the way it does.

We need a lightweight, versioned decision log that lives alongside the
code and is reviewed like code.

## Decision

We will keep Architecture Decision Records (ADRs) in `docs/adr/` using
the format popularized by Michael Nygard
([blog post](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions)).

- One Markdown file per decision, numbered sequentially from `0000`.
- Each ADR has sections: Context, Decision, Alternatives, Consequences.
- ADRs are immutable once Accepted. A later decision that changes course
  is a new ADR that marks the old one as Superseded.
- The index in `docs/adr/README.md` is updated in the same PR that
  introduces the ADR.

An ADR is required for changes to:

- package or module boundaries,
- the command or processing-element extension model,
- the error / exit-code model,
- the data-processing pipeline shape (in-memory vs streaming, spill policy),
- the release or support policy.

Smaller refactors and bug fixes do not need an ADR.

## Alternatives considered

- **Wiki pages.** Rejected: not versioned with the code, easy to drift.
- **Long PR descriptions only.** Rejected: not discoverable once the PR
  is merged and scrolled past.
- **RFC process with numbered proposals and formal voting.** Rejected as
  too heavy for a project of this size.

## Consequences

- Contributors to significant architectural changes must write an ADR as
  part of the PR. Reviewers may block a PR on a missing ADR.
- The repository gains a growing but bounded set of short decision
  records that future maintainers can read in order.
- There is no tooling requirement; ADRs are plain Markdown.
