# Documentation

Architecture and reference documentation for the English Active BYOK TOEIC Writing practice app.

## Navigation

### Architecture Decisions (ADRs)

| ADR | Topic | Status |
| --- | --- | --- |
| [ADR-001](decisions/ADR-001-app-scope.md) | App scope and constraints | Accepted |
| [ADR-002](decisions/ADR-002-ai-evaluation-reliability.md) | AI evaluation reliability strategy | Accepted |
| [ADR-003](decisions/ADR-003-framework-and-component-architecture.md) | Framework and component architecture | Accepted |
| [ADR-004](decisions/ADR-004-question-bank-and-data-portability.md) | Question bank and data portability | Accepted |
| [ADR-005](decisions/ADR-005-build-step-escape-hatch.md) | Build-step migration escape hatch | Proposed |
| [ADR-006](decisions/ADR-006-error-recovery-and-runtime-validation.md) | Error recovery and runtime validation | Accepted |

### Technical Notes (implementation companions)

- [Component contract](technical-notes/component-contract.md) — render/update/destroy lifecycle
- [File structure & load order](technical-notes/file-structure-and-load-order.md) — 14-file split, script tag ordering
- [Classic scripts, not ES modules](technical-notes/classic-scripts-not-es-modules.md) — why `file://` rules out `<script type="module">`
- [Cross-cutting concerns](technical-notes/cross-cutting-concerns.md) — AppState shape, rendering safety, CDN fallback
- [Model bakery](technical-notes/model-bakery.md) — baked-in model list mirror
- [Prompt assembly](technical-notes/prompt-assembly.md) — PROMPT_VERSION, per-Part prompt injection
- [Reliability implementation notes](technical-notes/reliability-implementation-notes.md) — Tier 1-3 technique details

### Reference Guides

- [Question schema](question-schema.md) — TypeScript discriminated union schema with per-Part rubric criteria
- [Model provider guide](model-provider-guide.md) — Gemini stable model list, temperature support, deprecation schedule
- [Reference library](reference-library.md) — ETS sources and calibration references

## Structure

```text
docs/
├── decisions/          # Architecture Decision Records (what was decided)
├── technical-notes/    # Implementation companions (how each ADR is implemented)
├── *.md                # Domain references (schemas, vendor guides, sources)
└── questions-sample.json  # Internal calibration (gitignored — not deployed)
```
