# File Structure and Load Order

> Technical reference — implementation guide for ADR-003 (file structure and load order).

## Decision

Split the application into focused files, each wrapping its contents in an IIFE and attaching only its public surface to `App.*`.

## File List

| File | Exports to `App.*` | Purpose |
| --- | --- | --- |
| `scripts/store.js` | `App.store` | `AppState` singleton (`get()` / `set()` / `subscribe()`) |
| `scripts/exportImport.js` | `App.exportImport` | `serialize()` / `deserialize(json)` — JSON export/import + version migration |
| `scripts/schema.js` | `App.schema` | Scoring bands, rubric criteria, `validateQuestion()` |
| `scripts/models.js` | `App.models` | Baked-in stable model list + temperature-support map |
| `scripts/questions.js` | `App.questions` | `loadQuestions()` loader (ADR-004 Phase 1/2) |
| `scripts/fewShotAnchors.js` | `App.fewShotAnchors` | Per-Part few-shot calibration anchors |
| `scripts/promptBuilder.js` | `App.promptBuilder` | Assembles system + evaluation prompts per Part |
| `scripts/evaluator.js` | `App.evaluator` | `evaluateAnswer()` + `handleApiError()` |
| `scripts/components/modelDropdown.js` | `App.components.ModelDropdown` | Model selection dropdown |
| `scripts/components/questionSelector.js` | `App.components.QuestionSelector` | Part → question selector |
| `scripts/components/answerEditor.js` | `App.components.AnswerEditor` | Answer textarea + word counter |
| `scripts/components/modelRecoveryBanner.js` | `App.components.ModelRecoveryBanner` | Dead-model recovery UI |
| `scripts/components/examinerOutput.js` | `App.components.ExaminerOutput` | Evaluation output rendering |
| `scripts/app.js` | (top-level controller) | Wires `<script defer>` tags, mounts components, calls `App.questions.loadQuestions()` at init |

## Load Order

Classic scripts share one global execution context and run in document order. `index.html` wires them by hand with a tiny inline bootstrap before any deferred script:

```html
<script>window.App = { components: {} };</script>
<script src="scripts/store.js" defer></script>
<script src="scripts/exportImport.js" defer></script>
<script src="scripts/schema.js" defer></script>
<script src="scripts/models.js" defer></script>
<script src="scripts/questions.js" defer></script>
<script src="scripts/fewShotAnchors.js" defer></script>
<script src="scripts/promptBuilder.js" defer></script>
<script src="scripts/evaluator.js" defer></script>
<script src="scripts/components/modelDropdown.js" defer></script>
<script src="scripts/components/questionSelector.js" defer></script>
<script src="scripts/components/answerEditor.js" defer></script>
<script src="scripts/components/modelRecoveryBanner.js" defer></script>
<script src="scripts/components/examinerOutput.js" defer></script>
<script src="scripts/app.js" defer></script>
```

This 15-line block in `index.html` *is* the dependency graph. The inline bootstrap runs at parse time, so `App` and `App.components` exist by the time `store.js` (the first deferred file) executes. `defer` guarantees this list runs, in this order, after HTML parsing completes and before `DOMContentLoaded`, without blocking render.

## Rationale

- `models.js` and `fewShotAnchors.js` are separated to keep vendor/config data and copyright-adjacent content out of the ADR body, following ADR-002's principle that vendor-specific facts live in a guide, not in an immutable ADR.
- `exportImport.js` is a dedicated persistence layer — it owns the serialize/deserialize logic and the version-migration function that ADR-004 requires, keeping `store.js` focused on live state management and cross-reference validation (ADR-006) separate from data-format concerns.
- It loads immediately after `store.js` so it can read `App.store.get()` for serialization and write back via `App.store.set()` on deserialization, but before `questions.js` so the question bank is available for `selectedPart`/`selectedQuestionId` cross-references during import.

## Cross-reference

- **ADR-003 D1:** Vanilla JS decision (primary)
- **ADR-003:** Classic `<script defer>` not ES modules (decision)
- **ADR-006:** Init-time and import-time validation call sites
- **ADR-004:** Storage, Export/Import schema versioning
- **ADR-005:** Build-step escape hatch trigger (script count exceeds 15)
