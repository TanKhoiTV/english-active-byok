# ADR-002: AI Evaluation Reliability — Rubric-Based Scoring Without a Backend

## Status

Accepted

## Date

2026-08-15

## Context

ADR-001 defines the app as a single-page, client-side TOEIC Writing practice tool (no backend, no telemetry, BYOK, GitHub Pages deployment). This ADR defines how the AI evaluation can be made reliable within those constraints.

TOEIC Writing consists of three task types, each scored on a different band scale by certified ETS human raters:

| Questions | Task | ETS Official Evaluation Criteria | Band Scale |
| :--- | :--- | :--- | :--- |
| 1–5 | Write a sentence based on a picture | Grammar, task completion, relevance to pictures | 0–3 |
| 6–7 | Respond to a written request | Sentence quality and variety, vocabulary, organization | 0–4 |
| 8 | Write an opinion essay | Opinion supported with reasons/examples, grammar, vocabulary, organization | 0–5 |

[CITATION C4, p. 5: ETS TOEIC Speaking & Writing Score User Guide — official criteria per Part as shown above.]

The core tension: LLM output is non-deterministic, and this app has none of the usual levers for taming that variance:

- No backend — cannot log or monitor score drift over time.
- No fine-tuning — cannot train a model to be more consistent.
- No telemetry — cannot see when a model update silently changes scoring behavior.
- No answer keys — TOEIC Writing is scored holistically by human raters; there is no fixed answer key to match against. Feedback is AI-generated against rubrics.
- BYOK — the user pays per request; multi-sample strategies have real cost implications.

[CITATION C4, p. 8: ETS TOEIC Speaking & Writing Score User Guide — "The Writing test responses are scored by certified ETS raters."]

All reliability must therefore be solved in the prompt and in the client, at request time. No single technique is sufficient — the strategy is a stack of independent, complementary techniques.

## Decision

Adopt a layered reliability approach with techniques prioritized by leverage-per-effort. Each technique is independently applied — if one fails, the others still constrain the output.

### Tier 1 — Required (high leverage, low effort)

#### 1. Criterion decomposition via structured output

**Problem**: The biggest source of variance in LLM grading isn't the rubric text — it's that a model asked for "a holistic score" jumps straight to a number without structured reasoning. Criterion decomposition forces the model to score each ETS-defined rubric dimension separately and derive the final score from those subscores.

**Solution**: Use the provider's structured-output mechanism (response schema enforcement) so the model cannot produce malformed or unstructured output. This is a mainstream capability across major providers — [see Model Provider Guide for the current provider's specific mechanism].

The JSON schema decomposes each Part's ETS criteria into separate scored sub-dimensions [CITATION C4, p. 5]:

- **Part 1 (0–3)**: grammar, task_completion, picture_relevance
- **Part 2 (0–4)**: sentence_quality_variety, vocabulary, organization
- **Part 3 (0–5)**: thesis_and_support, grammar, vocabulary, organization

This cuts variance more than almost anything else because it constrains how the model arrives at the number, not just the rubric text. It is also the technique least dependent on provider-specific parameters (like temperature) — schema enforcement works regardless of what other controls the provider offers. [CITATION-NOTE: Verify at implementation time that the chosen provider supports structured JSON schema output via the guide.]

#### 2. Determinism strategy — provider-adaptive

**Problem**: Different providers (and different model generations within the same provider) offer different levers for reducing output variance. A blanket configuration that works for one may fail or be rejected by another.

**Solution**: The app uses a two-layer determinism strategy:

1. **System instructions with explicit determinism rules** (universal): The system prompt must contain explicit rules for scoring. Example:
   > "Score deterministically. Do not add creative interpretation. Base every subscore strictly on the ETS criteria defined for this Part. If a criterion is not met at all, score 0 for that criterion."

2. **Schema enforcement** (Technique #1 above) — constrains the output format structurally, preventing the model from skipping the criterion-by-criterion reasoning step.

3. **Temperature (conditional, secondary)**: Where the provider's API and the current model support a temperature-like parameter, set it low (0.2) as an additional safety net. Where temperature is deprecated or ignored, rely on layers 1 + 2 alone.

[CITATION-NOTE: The specific provider's current temperature support is documented in the Model Provider Guide, not here, because it changes across model generations.]

#### 3. Pin exact model versions + handle dead-model recovery

**Problem**: Rolling aliases (e.g., "latest") change silently. Using an alias means the user's scoring behavior can shift without notice. Conversely, pinned models get shut down, and an imported/exported JSON settings file may reference a model that no longer exists.

**Solution**: Two policies:

1. **Pin to concrete model identifiers.** The app pins to specific model versions, never to rolling aliases. This is a living checklist item — the list of currently-stable models must be re-verified before each release. [See Model Provider Guide.]

2. **Dead-model recovery.** Since users import/export settings via JSON (ADR-001 scope), an imported JSON may reference a model that has since been shut down. The app must:
   - Catch provider API errors (HTTP 4xx for invalid/missing model) on `evaluateAnswer()`.
   - Display a user-friendly message: "The model in your exported settings is no longer available. Please select a different model above."
   - Pre-select the most recent stable model in the dropdown for one-click recovery.

[CITATION-NOTE: The current stable model list and deprecation/sunset schedule live in the Model Provider Guide. They must be re-pulled at implementation time — they are guaranteed stale by draft-time.]

#### 4. Present scores as approximate with provenance

- Display scores with a visible **"AI estimate, not an official TOEIC score"** disclaimer rather than bare numbers.
- Include provenance fields in every evaluation response: `model_used`, `prompt_version`, `timestamp`.
- If the prompt or model version changes, old and new scores are not treated as silently comparable — the provenance tags make the change visible in exported JSON.

### Tier 2 — Worth the one-time effort (medium effort, high value)

#### 1. Few-shot calibration anchors

Embed 1–2 worked examples per Part in the system prompt — a sample response with its criterion-by-criterion scoring, spanning a low and high band. This anchors the model's internal sense of scale.

[CITATION C6: Zhao, Y. et al. "On the (In)compatibility of Prompt Based and Prediction Based Approaches to Few-Shot Learning." ACL 2021. https://proceedings.mlr.press/v139/zhao21c.html; Brown, T. et al. "Language Models are Few-Shot Learners." arXiv:2005.14165 (2020). https://arxiv.org/abs/2005.14165]

[CITATION-NOTE: Examples built from TOEIC prep materials, paraphrased (not reproduced verbatim) for copyright safety.]

#### 2. Optional self-consistency (multi-sample vote)

Running the evaluation 3× and taking the median is architecturally simple (3 client-side fetches), but not cost-free. [CITATION C7: Wang, X.; Wei, J.; Schuurmans, D. et al. "Self-Consistency: Relying on Language Models to Improve Chain-of-Thought Reasoning in Language Models." arXiv:2203.11171 (2022). 704 citations. https://arxiv.org/abs/2203.11171]

Implemented as an opt-in **"High-confidence mode" toggle**. If the 3 scores disagree by more than 1 band:
[CITATION-NOTE: Define threshold as absolute ±1 band (simple to communicate) or relative ±20% of max (scale-normalized). Resolve before coding.]

- Surface: **"Scores varied — treat this as uncertain"** rather than silently averaging.

#### 3. Dev-time calibration testing

Build a small, fixed set of sample responses per Part with expected band scores. Run them through the prompt manually whenever the prompt or model version changes. This is a developer-only exercise — never runs in the user's browser.

[CITATION-NOTE: This is analogous to running known-answer test items against a scoring pipeline to verify stability — conceptually similar to how ETS validated its own test forms and rater training against field-study data. [CITATION C4, p. 6]
[CITATION-NOTE: Consider automating via CI; defer to v2.]

### Tier 3 — v2 / future (high effort, lower marginal value)

1. Server-side logging and drift detection — requires a backend. Rejected by ADR-001.
2. Model fine-tuning — violates BYOK and no-backend. Rejected by ADR-001.
3. Prompt A/B testing in UI — defer. May be useful if prompt versions diverge.

## Alternatives Considered

### A1. Simple holistic scoring (status quo)

- Rejected — highest variance; no structured reasoning.

### A2. Server-side logging and drift detection

- Rejected — violates no-backend and no-telemetry constraints (ADR-001).

### A3. Model fine-tuning

- Rejected — violates BYOK and no-backend constraints (ADR-001).

### A4. Answer key matching

- Rejected — TOEIC Writing is scored holistically by human raters; no fixed answer key exists. [CITATION C4, p. 8]

### A5. Fixed temperature = 0 (absolute determinism)

- Originally considered as a blanket policy, but now **rejected as vendor-specific**. Some providers/gen model generations do not support a temperature-like parameter, or treat it as deprecated/ignored. Relying on temperature alone would couple the ADR to a single provider's current API surface.

- The determinism strategy instead uses: **system instructions** (universal) + **schema enforcement** (universal) + **temperature as conditional secondary layer** where supported. [See Model Provider Guide for the current provider's temperature support status.]

## Priority Sequence

| Priority | Technique | Effort | Impact | Tier |
| :--- | :--- | :--- | :--- | :--- |
| 1 | Criterion decomposition (structured output) | Low | High | Tier 1 |
| 2 | Determinism strategy (system instructions + schema + conditional temperature) | Low | High | Tier 1 |
| 3 | Pin exact models + dead-model recovery | Medium | Medium | Tier 1 |
| 4 | Approximate scores + provenance tags | Low | Medium | Tier 1 |
| 5 | Few-shot calibration anchors (one-time) | Medium | High | Tier 2 |
| 6 | Optional self-consistency (3-sample vote) | High | Medium | Tier 2 |
| 7 | Dev-time calibration test set (one-time) | Medium | Medium | Tier 2 |

## Consequences

### Positive

- **Consistency**: Criterion decomposition + schema enforcement + system-instruction determinism significantly reduces score variance. The approach adapts to the provider's current API surface rather than breaking when APIs evolve.
- **Transparency**: Structured criteria output shows the user why a score was given. `error_analysis`, `polished_revision`, and `key_recommendations` fields provide actionable feedback.
- **Trust**: "AI estimate" framing + provenance tags + uncertainty flags set appropriate expectations.
- **Future-proofing**: Provenance tags (`model_used`, `prompt_version`, `timestamp`) let the app detect prompt/model changes. Dead-model recovery prepares for deprecation cycles. Provider-agnostic design allows switching providers without rewriting the ADR.
- **No constraint violations**: All Tier 1 and Tier 2 techniques are client-side only.

### Negative

- **Prompt complexity**: System instructions must carry explicit determinism rules. Must be versioned with `prompt_version`.
- **Model maintenance burden**: The stable-model list and temperature-support map must be re-verified before each release. This is now an explicit policy (Tier 1 #3), not a one-time task.
- **Cost for high-confidence mode**: Users pay 3× per evaluation. Mitigated by opt-in only.
- **Calibration maintenance**: Few-shot examples and dev-time test sets require periodic review.
- **Dual-response payload**: A single JSON response must carry both structured scores and human-readable text fields — increases schema complexity.

### Neutral

- The prompt/rubric version is embedded in `index.html`'s `SYSTEM_PROMPT` constant. Versioning requires a new release.
- Provenance tags are returned in the API response but not persisted in localStorage in v1. Export (ADR-001, in scope) naturally carries provenance data. Persistent history deferred.
- Self-consistency mode requires a UI toggle and additional API call management.
- Dead-model recovery requires catching API errors and presenting a reselection prompt — a UX consideration.
- Vendor-specific configuration (model IDs, URLs, deprecation dates) lives in `docs/model-provider-guide.md`, not in this ADR. This is intentional — see § Why this ADR is vendor-agnostic.

## Model Provider Guide

`docs/model-provider-guide.md` contains the concrete, frequently-changing vendor configuration:

- Current stable model list and identifiers (pinned versions)
- Temperature support status per model
- Deprecation and sunset schedule
- Structured output mechanism details (API endpoint, response schema config)
- Alias naming convention and examples

**Update rule**: Re-verify and update the guide before each new app release, using Tier 1 #3 as the trigger. The ADR's decisions remain valid as long as the guide is kept current.

### Why this ADR is vendor-agnostic (and a separate guide exists)

Vendor-specific configuration — exact model identifiers, API endpoint URLs, deprecation schedules, and which API parameters are currently supported — changes far more often than the reliability strategy itself. ADRs are governance artifacts: once accepted, they are superseded by new ADRs, not edited in place. Vendor config, by contrast, must be edited constantly.

To resolve this document-type mismatch, this ADR states **evergreen policies and methods**, and delegates **concrete vendor facts** to `docs/model-provider-guide.md`. The update rule for the guide is tied to the same lifecycle trigger as model-pinning (Tier 1 #3 below): re-verify before each new app release. See § Model Provider Guide (below) for the boundary between what lives where.

## References

1. **C1** — Structured output (schema enforcement) as a mainstream provider capability: See [Model Provider Guide](../model-provider-guide.md) for current provider-specific mechanism.
2. **C4** — ETS TOEIC Speaking & Writing Score User Guide (criteria, scales, scoring method, proficiency descriptors): <https://www.ets.org/pdfs/toeic/toeic-speaking-writing-score-user-guide.pdf> ; <https://www.ets.org/pdfs/toeic/toeic-speaking-writing-score-descriptors.pdf>
3. **C6** — Few-shot learning instability & calibration: Zhao, Y. et al. "On the (In)compatibility of Prompt Based and Prediction Based Approaches to Few-Shot Learning." ACL 2021. <https://proceedings.mlr.press/v139/zhao21c.html> ; Brown, T. et al. "Language Models are Few-Shot Learners." arXiv:2005.14165. <https://arxiv.org/abs/2005.14165>
4. **C7** — Self-consistency: Wang, X.; Wei, J.; Schuurmans, D. et al. "Self-Consistency: Relying on Language Models to Improve Chain-of-Thought Reasoning in Language Models." arXiv:2203.11171 (2022). <https://arxiv.org/abs/2203.11171>

## Related

- **ADR-001**: App Scope — defines the BYOK, no-backend, no-telemetry, GitHub Pages deployment constraints this ADR operates within.
