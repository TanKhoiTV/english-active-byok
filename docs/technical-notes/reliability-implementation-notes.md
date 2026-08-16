# Reliability Implementation Notes

> Companion reference for **ADR-002: AI Evaluation Reliability**. Contains implementation detail that is too granular for the ADR body. Re-verify model-dependent configuration against `docs/model-provider-guide.md` before each release.

## Tier 1 — Required Techniques

### 1. Criterion decomposition via structured output

The model must score each ETS-defined rubric dimension separately and derive the final score from those subscores. This cuts variance more than almost anything else because it constrains how the model arrives at the number, not just the rubric text.

**Implementation:**

- Use the provider's structured-output mechanism (response schema enforcement) so the model cannot produce malformed or unstructured output.
- The JSON schema decomposes each Part's ETS criteria into separate scored sub-dimensions:

| Part | Range | Sub-criteria |
| ------ | ------- | ------------- |
| Part 1 | 0–3 | grammar, task_completion, word_usage |
| Part 2 | 0–4 | sentence_quality_variety, vocabulary, organization |
| Part 3 | 0–5 | thesis_and_support, grammar, vocabulary, organization |

- [CITATION-NOTE] Verify at implementation time that the chosen provider supports structured JSON schema output via `docs/model-provider-guide.md`.

### 2. Determinism strategy — provider-adaptive

Different providers (and different model generations within the same provider) offer different levers for reducing output variance.

**Implementation:**

1. **System instructions with explicit determinism rules** (universal): The system prompt must contain:
   > "Score deterministically. Do not add creative interpretation. Base every subscore strictly on the ETS criteria defined for this Part. If a criterion is not met at all, score 0 for that criterion."

2. **Schema enforcement** (Technique #1 above) — constrains the output format structurally, preventing the model from skipping the criterion-by-criterion reasoning step.

3. **Temperature (conditional, secondary):** Where the provider's API and the current model support a temperature-like parameter, set it low (0.2) as an additional safety net. Where temperature is deprecated or ignored, rely on layers 1 + 2 alone.
   - [CITATION-NOTE] The specific provider's current temperature support is documented in `docs/model-provider-guide.md`.
   - As of 2026-08-15: temperature is **deprecated and ignored** on Gemini 3.6 Flash and 3.5 Flash-Lite, and will cause HTTP 400 on future model generations. Determinism on Gemini 3.x relies on system instructions + `responseSchema` alone. Temperature 0.2 is only a secondary safety net on legacy-supporting models (Gemini 3.5 Flash, 3.5 Flash-Lite, 2.5 Flash/Flash-Lite).

### 3. Pin exact model versions + handle dead-model recovery

Rolling aliases (e.g., "latest") change silently — using an alias means the user's scoring behavior can shift without notice.

**Implementation:**

1. **Pin to concrete model identifiers.** Never use rolling aliases. This is a living checklist item — the list of currently-stable models must be re-verified before each release against `docs/model-provider-guide.md`.

2. **Dead-model recovery.** Since users import/export settings via JSON (ADR-001 scope), an imported JSON may reference a model that has since been shut down:
   - Catch provider API errors (HTTP 4xx for invalid/missing model) on `evaluateAnswer()`.
   - Display: "The model in your exported settings is no longer available. Please select a different model above."
   - Pre-select the most recent stable model in the dropdown for one-click recovery.
   - See **ADR-006** for the shared `handleApiError()` / `ModelRecoveryBanner` implementation.

### 4. Present scores as approximate with provenance

- Display scores with a visible **"AI estimate, not an official TOEIC score"** disclaimer.
- Include provenance fields in every evaluation response: `model_used`, `prompt_version`, `timestamp`.
- If the prompt or model version changes, old and new scores are not treated as silently comparable — provenance tags make the change visible in exported JSON.
- Uses `PROMPT_VERSION` constants for dynamic prompt assembly — see `docs/technical-notes/prompt-assembly.md`.

## Tier 2 — Few-shot calibration anchors

Few-shot examples in the system prompt reduce inter-model variance by anchoring the model to a consistent scoring style.

**Implementation:**

- Keep ~3–5 examples per Part, sourced from `docs/questions-sample.json` (gitignored, internal calibration only — never embedded in the app). Anchors live in that file's `calibrationAnchors` section as original, paraphrased `{ level, prompt, response, criteria[], rationale }` objects (copyright-safe per ADR-004 Quality Gate).
- Examples show one high, one mid, and one low score per criterion, with brief rationale.
- Refresh before each release to match the current prompt version.
- **Runtime wiring (opt-in):** `index.html` must never `fetch()` or reference `docs/questions-sample.json` (ADR-004). The user loads it locally via a file picker (`FileReader`); `loadCalibrationFile()` parses and validates it in-memory, `onCalibrationToggle()` enables it, and `buildCalibrationSection(part)` appends the Part's anchors to `SYSTEM_PROMPT` for that evaluation only. The content is never uploaded, embedded, or persisted.

## Tier 3 — Deferred opt-in self-consistency

Self-consistency (running multiple samples and taking the majority/median) is available as an opt-in for users who want maximum reliability over cost.

**Implementation (deferred):**

- Run 3 samples per evaluation when the user toggles "high reliability mode."
- Take the majority vote per criterion subscore (not just the final score).
- [CITATION-NOTE] The self-consistency threshold (±1 vs. ±20% per-score-band) is a pending implementation decision. [CITATION: Wang, Xuezhi et al. "Self-Consistency: Relying on Language Models to Improve Chain-of-Thought Reasoning." arXiv:2203.11171 (2022).]

## See Also

- **ADR-002** — The decision record this document supports.
- **ADR-006** — Shared error handling and dead-model recovery implementation.
- **`docs/model-provider-guide.md`** — Current stable model list, temperature support per model family, deprecation schedule.
