# ADR-002: AI Evaluation Reliability — Rubric-Based Scoring Without a Backend

## Status

Accepted

## Date

2026-08-15

## Context

ADR-001 defines the app as a single-page, client-side TOEIC Writing practice tool (no backend, no telemetry, BYOK, GitHub Pages deployment). This ADR defines how AI evaluation reliability can be achieved within those constraints.

TOEIC Writing consists of three task types, each scored on a different band scale by certified ETS human raters:

| Questions | Task | ETS Official Evaluation Criteria | Band Scale |
| :--- | :--- | :--- | :--- |
| 1–5 | Write a sentence based on a picture | Grammar, task completion, relevance to pictures | 0–3 |
| 6–7 | Respond to a written request | Sentence quality and variety, vocabulary, organization | 0–4 |
| 8 | Write an opinion essay | Opinion supported with reasons/examples, grammar, vocabulary, organization | 0–5 |

[CITATION C4, p. 5: ETS TOEIC Speaking & Writing Score User Guide — official criteria per Part.]

The core tension: LLM output is non-deterministic, and this app has none of the usual levers for taming that variance:

- No backend — cannot log or monitor score drift over time.
- No fine-tuning — cannot train a model to be more consistent.
- No telemetry — cannot see when a model update silently changes scoring behavior.
- No answer keys — TOEIC Writing is scored holistically by human raters; there is no fixed answer key to match against. Feedback is AI-generated against rubrics.
- BYOK — the user pays per request; multi-sample strategies have real cost implications.

[CITATION C4, p. 8: ETS TOEIC Speaking & Writing Score User Guide — "The Writing test responses are scored by certified ETS raters."]

All reliability must therefore be solved in the prompt and in the client, at request time. No single technique is sufficient — the strategy is a stack of independent, complementary techniques.

## Decision

Adopt a **layered reliability approach** with techniques prioritized by leverage-per-effort, all constrained to client-side execution only. The strategy stacks four Tier 1 required techniques — criterion decomposition via structured output, provider-adaptive determinism (system instructions + schema enforcement + temperature where supported), exact model version pinning with dead-model recovery, and approximate-score presentation with provenance tags — with Tier 2–3 techniques (few-shot calibration anchors, self-consistency voting) deferred as optional optimizations.

The full implementation detail for each technique (including provider-specific temperature support, structured-output schema examples, dead-model recovery flow, and provenance tag formats) lives in `docs/technical-notes/reliability-implementation-notes.md` to keep this ADR focused on the decision, not the specification.

## Alternatives Considered

### Holistic grading without decomposition (status quo)

- **Rejected** — highest variance; the model jumps straight to a number without structured reasoning. No constraint satisfied.

### Server-side logging and drift detection

- **Rejected** — violates the no-backend and no-telemetry constraints (ADR-001). Cannot log or monitor score drift over time.

### Model fine-tuning

- **Rejected** — violates BYOK and no-backend constraints (ADR-001). Cannot train a model to be more consistent.

### Answer key matching

- **Rejected** — TOEIC Writing is scored holistically by human raters; no fixed answer key exists. Matching against any "answer key" would produce misleading feedback. [CITATION C4, p. 8]

### Fixed temperature = 0 (absolute determinism)

- **Rejected as vendor-specific.** Some providers/generations do not support a temperature-like parameter, or treat it as deprecated and ignored (as of the July 21 2026 API changes, temperature/top_p/top_k are deprecated across the Gemini suite and will cause HTTP 400 on future model generations). Relying on temperature alone would couple this ADR to a single provider's current API surface instead of remaining provider-agnostic.
- The adopted determinism strategy instead layers: **system instructions** (universal) + **schema enforcement** (universal) + **temperature as a conditional secondary layer** where the provider supports it. See `docs/model-provider-guide.md` for the current temperature support status per model.

## Consequences

### Positive

- **Consistency**: Criterion decomposition + schema enforcement + system-instruction determinism significantly reduces score variance. The provider-adaptive design adapts to the current API surface rather than breaking when APIs evolve.
- **Transparency**: Structured criteria output shows the user why a score was given. `error_analysis`, `polished_revision`, and `key_recommendations` fields provide actionable feedback.
- **Trust**: "AI estimate" framing + provenance tags + uncertainty flags set appropriate expectations.
- **Future-proofing**: Provenance tags (`model_used`, `prompt_version`, `timestamp`) let the app detect prompt/model changes. Dead-model recovery prepares for deprecation cycles. Provider-agnostic design allows switching providers without rewriting this ADR.
- **No constraint violations**: All Tier 1 techniques are client-side only.

### Negative

- **Prompt complexity**: System instructions must carry explicit determinism rules. Must be versioned with `PROMPT_VERSION` (see `docs/technical-notes/prompt-assembly.md`).
- **Model maintenance burden**: The stable-model list and temperature-support map must be re-verified before each release. This is now an explicit policy (Tier 1 #3), not a one-time task. See `docs/model-provider-guide.md`.
- **Duplicate-response payload**: A single JSON response must carry both structured scores and human-readable text fields — increases schema complexity.
- **Dead-model recovery requires catching API errors and presenting a reselection prompt** — a UX concern (ADR-006).

### Neutral

- Tier 2+ optimization techniques (few-shot calibration anchors, self-consistency voting, dev-time calibration testing) are documented in `docs/technical-notes/reliability-implementation-notes.md` as implementation notes, not decisions.
- The prompt/rubric version is embedded in `index.html`'s `SYSTEM_PROMPT` constant. Versioning requires a new release.
- Provenance tags are returned in the API response and held in the in-memory `answerHistory` field of `AppState` (see `docs/technical-notes/cross-cutting-concerns.md`) for the current session. They are **not persisted to localStorage** in v1 — the page starts with an empty history on each reload. Export (ADR-001, in scope) serializes whatever history exists in memory at export time, carrying provenance data natively. Persistent (cross-reload) history is deferred.
- Vendor-specific configuration (model IDs, URLs, deprecation dates) lives in `docs/model-provider-guide.md`, not in this ADR. This is intentional: ADRs are immutable and superseded by new ADRs, whereas vendor config churns in place and must be edited. The guide is re-verified before each release, using Tier 1 #3 (model pinning) as the trigger.

## Model Provider Guide

`docs/model-provider-guide.md` contains the concrete, frequently-changing vendor configuration:

- Current stable model list and identifiers (pinned versions)
- Temperature support status per model
- Deprecation and sunset schedule
- Structured output mechanism details (response schema config)
- Alias naming convention and examples

**Update rule**: Re-verify and update the guide before each new app release, using Tier 1 #3 (model pinning) as the trigger. This ADR's decisions remain valid as long as the guide is kept current.

## References

1. **C1** — Structured output (schema enforcement) as a mainstream provider capability: See `docs/model-provider-guide.md` for current provider-specific mechanism.
2. **C4** — ETS TOEIC Speaking & Writing Score User Guide (criteria, scales, scoring method, proficiency descriptors): <https://www.ets.org/pdfs/toeic/toeic-speaking-writing-score-user-guide.pdf> ; <https://www.ets.org/pdfs/toeic/toeic-speaking-writing-score-descriptors.pdf>
3. **C6** — Few-shot learning instability & calibration: Zhao, Y. et al. "On the (In)compatibility of Prompt Based and Prediction Based Approaches to Few-Shot Learning." ACL 2021. <https://proceedings.mlr.press/v139/zhao21c.html> ; Brown, T. et al. "Language Models are Few-Shot Learners." arXiv:2005.14165. <https://arxiv.org/abs/2005.14165>
4. **C7** — Self-consistency: Wang, X.; Wei, J.; Schuurmans, D. et al. "Self-Consistency: Relying on Language Models to Improve Chain-of-Thought Reasoning in Language Models." arXiv:2203.11171 (2022). <https://arxiv.org/abs/2203.11171>

## Related

- **ADR-001** — App scope: defines the BYOK, no-backend, no-telemetry, GitHub Pages deployment constraints this ADR operates within.
- **ADR-003** — Framework and component architecture — the vanilla JS / no-build decision that constrains all Tier 1 techniques to client-side execution.
- **ADR-004** — Question bank and data portability — import-time dead-model check (orthogonal to this ADR's runtime recovery in ADR-006); rubric criteria source for criterion decomposition.
- **ADR-005** — Build-step escape hatch: if triggered, may affect how Tier 2 optimizations (self-consistency voting) are implemented.
- **ADR-006** — Error recovery and runtime validation: implements Tier 1 #3 (dead-model recovery) and Tier 1 #4 (provenance/prompt versioning) as shared client-side components.
