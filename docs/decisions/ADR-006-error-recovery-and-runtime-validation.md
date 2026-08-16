# ADR-006: Error Recovery and Runtime Validation Strategy

## Status

Accepted

## Date

2026-08-16

## Context

Two implementation concerns need architectural decisions, both flowing from ADR-003's vanilla-JS constraint:

1. **Error handling:** API failures must be caught, classified, and presented to the user. ADR-002 Tier 1 #3 requires dead-model recovery at runtime (`evaluateAnswer()` catching HTTP 400/404), and ADR-004 requires a complementary import-time check. Both must use the same UI and message to avoid confusing users.
2. **Runtime validation:** `docs/question-schema.md` defines TypeScript types for question objects, but browser-executed JS ignores types at runtime. Questions are loaded from embedded JSON (Phase 1) or fetched JSON (Phase 2) — neither is type-checked by the browser.

## Decision

Centralize all API error handling through one `handleApiError(error, response)` function. Dead-model recovery is handled by one `ModelRecoveryBanner` component with two trigger points (runtime per ADR-002; import-time per ADR-004). Runtime validation uses one hand-rolled JavaScript validator with two call sites: crash at init-time on dev mistakes, degrade gracefully at import-time on user data.

## Alternatives Considered

### Shared handler vs. per-component error handling

A per-component approach would attach a try/catch to every `fetch()` call site, each with its own error message and recovery flow. This was rejected: it produces inconsistent messages, no dead-model recovery consolidation, and makes it impossible to add a new recovery behavior (e.g., "show model reselection banner") without editing every call site.

### Hand-rolled validator vs. zod/ajv

- **zod / ajv runtime validation.** Rejected: adds a dependency and/or build step, violating ADR-003 D1.
- **TypeScript `tsc` runtime validation.** Not viable: no build step, browser executes JS directly.
- **Hand-rolled validator.** Accepted: ~30 lines covering the question schema's depth, zero dependencies, crashes on developer mistakes at dev-time and degrades gracefully on user data at import-time.

## Call Sites

### Init-time (dev guard)

Every question in the embedded bank is validated at startup (Phase 1 inline JSON or Phase 2 `questions.json`). A failure **throws** before render — this catches authoring bugs, not user error. The validator runs during `loadQuestions()` before the first `render()`.

### Import-time (user data cross-reference)

User-imported JSON does **not** contain question objects — ADR-004's Export/Import table explicitly excludes the question bank from user export. On import, `App.exportImport.deserialize()` (owned by `scripts/exportImport.js`):

1. Parses the JSON and checks the `version: 1` field (runs migration function if a newer version is detected).
2. Validates `selectedModel` against `App.models.STABLE_MODELS`. If the model is deprecated, triggers `ModelRecoveryBanner.showModelRecovery(...)` (the import-time dead-model check required by ADR-004).
3. Cross-references the user's `selectedPart` / `selectedQuestionId` against the current in-memory question bank. If the referenced question no longer exists, resets the selector to a default Part/Question and surfaces an informational message.

This is a reference-resolution check, not a per-question validation — there are no question objects in the imported JSON to validate.

## Consequences

### Positive

- One UX for dead-model recovery across both triggers (runtime + import-time).
- Four error classes (429, 400/404, 401/403, network) map to one actionable message each; raw API errors never surface.
- Dev-time crashes catch authoring bugs before users see them; import-time graceful degradation prevents data loss.
- No validation dependency — consistent with the no-framework, no-build constraint.

### Negative

- The hand-rolled validator must be kept in sync with `docs/question-schema.md` by manual discipline.
- `handleApiError()` is a single point of failure — changes affect every fetch call site.
- `ModelRecoveryBanner` has two trigger points in different files — both must stay coordinated.

### Neutral

- Both call sites are owned by different files (ADR-003 file list): `evaluator.js` for runtime, `exportImport.js` for import-time. The ownership split prevents coupling but requires the ADR to document the contract.
- JSON export includes `apiKey` as opt-in; all data is tagged `version: 1` for future schema migration.

## References

1. **ADR-003 D1** — Vanilla JS, no framework, no build step: the constraint that ruled out zod/ajv.
2. **ADR-002** — AI evaluation reliability: Tier 1 #3 (dead-model recovery → runtime trigger), Tier 1 #4 (provenance tags).
3. **ADR-004** — Question bank and data portability: Phase 1/2 storage, import-time dead-model check (→ this ADR), and the `file://` support goal that ruled out ES modules.
4. **ADR-005** — Build-step escape hatch: the future decision that could adopt a compiled validator if the no-build constraint is superseded.
5. **C1** — `docs/question-schema.md`: TypeScript discriminated union (`QuestionPart1` | `QuestionPart2` | `QuestionPart3`), `RubricCriteriaPart1/2/3` literal types, and `validateQuestion()` pseudocode. ADR-006 ports this to a runtime validator.
6. **C2** — `docs/model-provider-guide.md`: source of truth for `STABLE_MODELS` and `TEMPERATURE_SUPPORT` (checked against by the import-time validator).

## Related

- **ADR-002** — Tier 1 #3 drives the runtime dead-model trigger (this ADR). Tier 1 #4 drives prompt versioning (`docs/technical-notes/prompt-assembly.md`).
- **ADR-003** — D1 (vanilla JS) constrains the validator to be dependency-free. File structure (`docs/technical-notes/file-structure-and-load-order.md`) owns the call sites.
- **ADR-004** — Import-time dead-model check is the second trigger for `ModelRecoveryBanner` (this ADR).
