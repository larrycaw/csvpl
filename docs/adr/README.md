# Architecture Decision Records

This directory holds Architecture Decision Records (ADRs) for CSVPL.

An ADR captures a single significant architectural decision: the context,
the choice made, the alternatives considered, and the consequences. ADRs
are **immutable once accepted**; a later decision that changes direction
gets a new ADR that supersedes the earlier one.

## Format

Each ADR is a single Markdown file named `NNNN-short-title.md`, where
`NNNN` is a zero-padded sequence number. The format follows Michael
Nygard's original proposal (see `0000-record-architectural-decisions.md`).

## Index

| ID | Title | Status |
|---|---|---|
| [0000](0000-record-architectural-decisions.md) | Record architectural decisions | Accepted |

New ADRs are added by appending to this table in the same PR that introduces
the ADR.

## Lifecycle

- **Proposed** — drafted in a PR, awaiting review.
- **Accepted** — merged. Becomes immutable.
- **Superseded** — a later ADR replaces it; keep the file and add a
  "Superseded by ADR-NNNN" note at the top.
- **Rejected** — drafted but not merged, or merged as a record of a
  decision not to proceed.

## Writing a new ADR

Copy `0000-record-architectural-decisions.md` as a template, increment the
number, and open a PR. Keep ADRs short — if it takes more than a page, the
decision probably needs to be decomposed.
