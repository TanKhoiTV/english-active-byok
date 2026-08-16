# Contributing

Thanks for helping improve English Active BYOK. This guide explains how the code and the documentation relate, so contributions stay consistent.

## Implementation authority

The project currently runs as a single static page, `index.html`. The `docs/` folder holds the architecture decisions (ADRs) and technical notes that describe how the project *should* work.

- `index.html` is the **current implementation of record** — the only running code, deployed as-is to GitHub Pages. It is a prototype, not a throwaway demo.
- `docs/` (ADRs + technical notes) are the **authoritative spec** — decisions, schema shape, and reliability strategy.
- **Rule:** where `index.html` and the docs contradict or are ambiguous, the **docs win as design** and `index.html` should be brought into conformance (or the doc corrected if the doc is stale).
- The embedded question bank was deliberately placed in `index.html` as the canonical **data store** (per ADR-004 Phase 1); `docs/question-schema.md` is the authoritative **shape** for that data.

## Where things live

- `index.html` — the app and its embedded question bank.
- `docs/decisions/` — Architecture Decision Records (ADRs).
- `docs/technical-notes/` — implementation companions for each ADR.
- `docs/question-schema.md` — the canonical question JSON shape.
- `docs/toeic-format-reference.md` — per-Part TOEIC format reference.
- `docs/README.md` — documentation navigation.

## Editing questions

Questions are embedded as JSON in `index.html` (ADR-004 Phase 1). Follow `docs/question-schema.md` for the shape and `docs/toeic-format-reference.md` for the per-Part format. Questions must be original (no copyrighted ETS content) and pass a copy-editing and human review pass.
