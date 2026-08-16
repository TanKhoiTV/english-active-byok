# Roadmap

Single source of truth for **next objectives** — deferred or not-yet-implemented work. Derived from the ADRs, which remain authoritative on *how* things should work. Governance: `index.html` is the current implementation of record; `docs/` is the authoritative spec (see `CONTRIBUTING.md`). When the two conflict, docs win as design.

## Current state

- `index.html` — monolithic single-page app; no build step (ADR-001, ADR-003 D1).
- Question bank — 9 embedded questions, ADR-004 Phase 1 (inline JSON in `index.html`).
- Evaluator — ADR-002 Tier 1 techniques implemented; per-Part content-coverage signals (`targetWords`, `directives`, `title`/`essayTopic`) wired into the examiner prompt via `buildQuestionText()`.

## Next objectives

### Structure (ADR-003 / ADR-005)

- **ADR-003 component split** — 14-file vanilla-JS architecture documented in `docs/technical-notes/file-structure-and-load-order.md`; **0/14 built**. Defer until load-order pain actually appears.
  - *Coupling:* the 14-file count sits just under ADR-005's 15-script-tag cliff, so doing the split will likely trigger an ADR-005 superseding decision. Do not schedule in isolation.
- **ADR-005 build-step escape hatch** — placeholder only; activates at >15 script tags + load-order bugs.

### Features

- **Model selection dropdown** — explicitly "planned" in README; absent from `index.html`. `docs/model-provider-guide.md` is already structured for future providers.
- **Question-bank growth (ADR-004)** — Phase 2 (`questions.json` + `fetch`/cache) only if the bank exceeds ~30 questions / 50 KB. New questions: human-curated originals + copy-editing gate; export/import is additive; user-custom questions deferred to a future ADR.
- **Persistent cross-reload history** — deferred in ADR-002 (v1 history held in memory only).
- **Reliability Tier 2–3** — few-shot calibration anchors (Tier 2), self-consistency voting (Tier 3, deferred opt-in); optional optimizations, not built.

### Docs / correctness

- ADR-002 ↔ content-coverage cross-reference — **done** (see git log). No open doc gaps known.

## Watch list

- `docs/reference-library.md` → "Future Sources" — research items to track.
