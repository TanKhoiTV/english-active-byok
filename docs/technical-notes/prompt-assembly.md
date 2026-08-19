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

## Prompt version history

Bumped on any prompt change (per ADR-002 Tier 1 #4); the constant is returned in every evaluation's `provenance.prompt_version` so results are traceable to the exact prompt.

- **1.1.0** (2026-08-16) — Bumped to reflect the Part 2 directive standardization and rubric alignment to ETS 0–4 holistic scoring (commits `9e80a93` → `0e54f61`). These prompt changes originally shipped under the `1.0.0` label due to a versioning oversight; `1.1.0` retroactively corrects the provenance label so post-refactor evaluations are traceable to the standardized Part 2 prompt.
- **1.0.0** — Initial prompt baseline. Note: this label also covered the post-Part-2-refactor prompt because the version was not advanced at refactor time (see 1.1.0).

## Cross-reference

- **ADR-002:** Criterion decomposition (Tier 1 #1), provenance tags (Tier 1 #4), few-shot anchors (Tier 2)
- **ADR-003 D1:** Vanilla JS (no monolithic SYSTEM_PROMPT constant)
