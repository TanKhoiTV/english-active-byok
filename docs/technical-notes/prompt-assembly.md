# Prompt Assembly with Version Tracking

> Technical reference — implementation guidance for ADR-003 (prompt assembly).

## Decision

Replace the monolithic `SYSTEM_PROMPT` constant with a `promptBuilder` module that assembles the system prompt dynamically per Part.

## Injected Components

The builder injects the following into each Part's system prompt:

1. **Scoring band** for the current Part — from `App.schema` (sourced from `docs/question-schema.md`)
2. **Rubric criteria** per Part — from `App.schema.RUBRIC_CRITERIA`, itself sourced from `docs/question-schema.md` and aligned to official ETS sub-criteria (ADR-002 Context, verified against the ETS Score User Guide)
3. **Few-shot calibration anchors** per Part — from `App.fewShotAnchors`, paraphrased for copyright safety per ADR-004's Quality Gate section
4. **`PROMPT_VERSION` constant** — bumped on any prompt change, returned in the evaluation `provenance` field per ADR-002 Tier 1 #4

## Rationale

ADR-002 requires that criterion decomposition and few-shot anchors are applied per Part, not globally. Hardcoding these into one monolithic prompt makes versioning and per-Part differentiation impossible. The `PROMPT_VERSION` constant creates an audit trail: every evaluation result records which prompt version produced it, enabling future drift detection or prompt iteration without silent regressions.

## Cross-reference

- **ADR-002:** Criterion decomposition (Tier 1 #1), provenance tags (Tier 1 #4), few-shot anchors (Tier 2)
- **ADR-003 D1:** Vanilla JS (no monolithic SYSTEM_PROMPT constant)
