# ADR-003: Framework and Component Architecture for the Static-Only App

## Status

Proposed

## Date

2026-08-16

## Context

ADR-001 scopes the app as a single-page, client-side TOEIC Writing practice tool: single static HTML, no backend, no build pipelines, GitHub Pages, BYOK, offline-capable UI. The current `index.html` is vanilla JS (no build step) loading Tailwind, Marked.js, and DOMPurify from CDN, with a monolithic `SYSTEM_PROMPT` constant.

We are restructuring the flat inline script into a component-level architecture. Several decisions were explicitly deferred to this ADR by other documents and must be resolved here:

- **Framework choice** — does "no build pipelines" (ADR-001) permit a framework that bundles at dev time? This decision constrains every later one.
- **`QuestionSelector` + `loadQuestions()`** — referenced by ADR-001 and ADR-004; needs a defined data contract and defined loading/error UI states.
- **`validateQuestion()` execution** — `docs/question-schema.md` defines TypeScript types, but types do not validate JSON at runtime.
- **Prompt assembly** — ADR-002 defines per-Part rubric criteria and few-shot anchors; `SYSTEM_PROMPT` must inject them per Part and carry `prompt_version`.
- **Model dropdown** — `docs/model-provider-guide.md` owns the stable list and temperature-support map; the dropdown must use a baked-in copy, not a live API call.
- **Shared dead-model recovery** — ADR-002 (runtime 4xx) and ADR-004 (import time) both point at one implementation.
- **Rendering safety, CDN fallback, state shape, general API error handling** — cross-cutting concerns nothing else owns.

This ADR resolves all of them. The framework decision comes first because it determines the rest.

## Decision

### 1. Framework: stay vanilla JS — no framework, no build step

**Decision.** Retain **vanilla JavaScript with ES modules** as the only "framework." Do **not** adopt React, Vue, Alpine.js, Preact+htm, or Lit. Do **not** add a build step, and do **not** amend ADR-001's no-build constraint (rejecting option (c) from the analysis).

**Tradeoff.** ADR-001's "no build pipelines" is read as written. A dev-only build that emits committed static HTML (option (c)) would technically satisfy "static deploy," but it imports a toolchain, lockfile churn, and a CI dependency for an app whose entire dataset is a small embedded question bank — a cost with no offset at this scale. CDN-loadable frameworks (option (b): Alpine, Preact+htm, Lit) avoid the build but still add a dependency and a reactivity model this app does not need (5–10 components, infrequent UI updates). Vanilla modules give decomposition with zero framework overhead and the smallest failure surface.

**Component model.** Components are self-contained module functions with a `render()` / `update(props)` / `destroy()` lifecycle, one per file. State flows one direction: `AppState` → props → `render()`. This is a minimal emulation of the framework component contract — enough to decompose features, portable to Alpine/Lit/React later if the app outgrows it.

### 2. File structure

A single static HTML page wired from multiple `<script type="module">` files. Storage of the question bank itself is ADR-004's call (inline JSON in `index.html` for Phase 1, `questions.json` for Phase 2); ADR-003 defines the *access point and contracts* around it.

```
/index.html                     — HTML skeleton + component wiring; embeds Phase 1 bank as
                                   <script type="application/json" id="question-bank"> (per ADR-004)
/scripts/
  store.js                      — AppState singleton (§10)
  schema.js                     — QuestionType constants + RUBRIC_CRITERIA + SCORING_BANDS + validateQuestion() (§3,§4,§5)
  models.js                     — STABLE_MODELS + TEMPERATURE_SUPPORT, mirrored from model-provider-guide.md (§6)
  questions.js                  — loadQuestions() loader (per ADR-004 Phase 1/2) (§3)
  fewShotAnchors.js             — per-Part few-shot anchors, paraphrased for copyright (§5)
  promptBuilder.js              — assembles SYSTEM_PROMPT per Part (§5)
  evaluator.js                  — evaluateAnswer() + handleApiError() (§7,§11)
  app.js                        — top-level controller; mounts components; calls loadQuestions() at init
  components/
    modelDropdown.js
    questionSelector.js
    answerEditor.js
    modelRecoveryBanner.js
    examinerOutput.js
```

`models.js` and `fewShotAnchors.js` are new modules that keep vendor/config data and copyrighted-adjacent content out of this ADR body, consistent with ADR-002's "vendor config lives in a separate doc" principle.

### 3. QuestionSelector contract + loadQuestions() integration

**Selection is lifted up.** `QuestionSelector` is presentational; it holds no selection state. It receives the data and the current selection, and reports changes upward:

```javascript
// scripts/components/questionSelector.js
export function QuestionSelector({ questions, value, onChange }) {
  // value: { part, questionId }; onChange(next) writes back to AppState
  return { render() {/* Part buttons + question dropdown */}, update(next) {}, destroy() {} };
}
```

**`loadQuestions()` is the single access point**, implemented per ADR-004's Phase 1/2 strategy:

```javascript
// scripts/questions.js
export async function loadQuestions() {
  const inline = document.getElementById('question-bank')?.textContent;
  if (inline) {                                  // Phase 1: embedded, no fetch
    try { return JSON.parse(inline).questions; }
    catch { throw new Error('Embedded question bank is corrupt.'); }
  }
  try {                                          // Phase 2: fetch + cache
    const data = await (await fetch('questions.json', { cache: 'no-cache' })).json();
    localStorage.setItem('questions_cache', JSON.stringify(data));
    return data.questions ?? data;
  } catch {
    const cached = localStorage.getItem('questions_cache');
    return cached ? (JSON.parse(cached).questions ?? JSON.parse(cached)) : FALLBACK_QUESTIONS;
  }
}
```

**Call site and UI states.** `app.js` calls `await loadQuestions()` once at init, stores the result in `AppState`, and passes `questions` to `QuestionSelector` via props.

- **Phase 1:** load is synchronous at parse; no loading UI. Failure (corrupt inline JSON) throws at init → fatal "question bank failed validation" message (a dev bug, never user data).
- **Phase 2:** while the fetch is pending, `QuestionSelector` renders disabled dropdowns / a skeleton. On network or cache miss, it falls back to `localStorage` then `FALLBACK_QUESTIONS` and shows a non-blocking "using offline copy" notice. The UI never blocks or crashes.

**Phase 1→2 is transparent to the component:** both phases return a `Question[]`, so `QuestionSelector` and `AppState` are unchanged by the migration.

### 4. validateQuestion(): hand-rolled runtime validator

**Decision.** A hand-rolled JS validator (no zod/ajv, no TypeScript compilation) is the runtime enforcer. `docs/question-schema.md`'s discriminated union is the design artifact; this is the enforcement. The validator is defined with a real `assert` and shared constants:

```javascript
// scripts/schema.js
export const SCORING_BANDS = { 1: 3, 2: 4, 3: 5 };

function assert(cond, msg) { if (!cond) throw new Error(`validateQuestion: ${msg}`); }

export function validateQuestion(q) {
  assert(q.part === 1 || q.part === 2 || q.part === 3, `invalid part ${q?.part}`);
  assert(typeof q.id === 'string' && q.id.startsWith(`p${q.part}-`), `id ${q?.id} violates p${q.part}-`);
  assert(q.scoringBand?.min === 0, 'scoringBand.min must be 0');
  assert(q.scoringBand?.max === SCORING_BANDS[q.part], `scoringBand.max must be ${SCORING_BANDS[q.part]}`);
  assert(Array.isArray(q.rubricCriteria) && q.rubricCriteria.length > 0, 'rubricCriteria required');
  if (q.part === 1) assert(Array.isArray(q.targetWords) && q.targetWords.length === 2, 'Part 1 needs 2 targetWords');
  if (q.part === 2) assert(q.emailContext != null, 'Part 2 needs emailContext');
  if (q.part === 3) assert(typeof q.essayTopic === 'string' && q.essayTopic.length > 10, 'Part 3 needs essayTopic');
  return true;
}
```

**Call sites.**

- *Init (dev guard):* every question in the embedded bank is validated at startup. A failure throws before render — this catches authoring bugs, not user error.
- *Import time (user data):* each question in an imported `questionBank` is validated inside a `try/catch`. Malformed questions are rejected with a user-facing message and skipped — never a crash, because user data crosses a trust boundary.

### 5. Prompt assembly: SYSTEM_PROMPT, prompt_version, per-Part criteria, few-shot

A `PromptBuilder` module (`scripts/promptBuilder.js`) assembles the system prompt dynamically instead of a monolithic constant:

```javascript
// scripts/promptBuilder.js
import { RUBRIC_CRITERIA, SCORING_BANDS } from './schema.js';
import { FEW_SHOT_ANCHORS } from './fewShotAnchors.js';

export const PROMPT_VERSION = '1.0';

export function buildSystemPrompt(part) {
  return `You are an expert ETS TOEIC Writing examiner...
Scoring band: ${SCORING_BANDS[part]}
Rubric criteria: ${RUBRIC_CRITERIA[part].join(', ')}
Few-shot examples:
${FEW_SHOT_ANCHORS[part]}`;
}

export function buildEvaluationPrompt(question, answer) {
  return `Question Prompt: ${question.prompt}
${question.emailContext ? `Email: ${JSON.stringify(question.emailContext)}\n` : ''}
${question.essayTopic ? `Essay Topic: ${question.essayTopic}\n` : ''}
Target words: ${question.targetWords?.join(', ') || 'N/A'}
${question.wordTarget ? `Word target: ${question.wordTarget.min}–${question.wordTarget.max} words\n` : ''}
Student Answer: ${answer}`;
}
```

- **Rubric criteria** come from `RubricCriteriaPart1/2/3` in `docs/question-schema.md`, exposed as the `RUBRIC_CRITERIA` constant in `schema.js` — never hardcoded strings.
- **Few-shot anchors** (ADR-002 Tier 2 #1) live in `fewShotAnchors.js`, one pair per Part, paraphrased for copyright safety.
- **`PROMPT_VERSION`** is a constant bumped on any prompt change and returned in the evaluation `provenance` fields per ADR-002 §Tier 1 #4.

### 6. Model dropdown: baked-in list from model-provider-guide.md

The dropdown renders from `STABLE_MODELS` and `TEMPERATURE_SUPPORT` in `scripts/models.js`, which is a **runtime mirror of `docs/model-provider-guide.md`**. The guide is the source of truth (human-readable, re-verified before each release per ADR-002 Tier 1 #3); `models.js` is the copy the UI imports. ADR-003 does not embed the list — it references the guide.

```javascript
// scripts/models.js — values mirrored from docs/model-provider-guide.md
export const STABLE_MODELS = [ /* { id, label } per guide */ ];
export const TEMPERATURE_SUPPORT = { /* id -> bool per guide */ };

// scripts/components/modelDropdown.js
import { STABLE_MODELS, TEMPERATURE_SUPPORT } from '../models.js';
export function getTemperature(modelId) {
  return TEMPERATURE_SUPPORT[modelId] ? 0.2 : undefined; // undefined -> omit from generationConfig
}
```

`getTemperature()` applies ADR-002's conditional-temperature determinism rule: set `0.2` only where the model supports it; otherwise omit the parameter.

**Offline by default.** The dropdown ships with the baked-in `STABLE_MODELS`. The provider guide's suggestion to call `GET …/v1beta/models` at *implementation* time is a maintainer instruction, not a runtime dependency — a live `models.list()` at runtime would fail offline and empty the dropdown, violating ADR-001. Live discovery, if ever added (v2 "refresh" button), would *augment* the baked-in list, never replace it.

### 7. Shared dead-model recovery banner: one component, two call sites

A single `ModelRecoveryBanner` component (`scripts/components/modelRecoveryBanner.js`) is defined once and triggered from two places:

```javascript
// scripts/components/modelRecoveryBanner.js
import { STABLE_MODELS } from '../models.js';
export function showModelRecovery(deprecatedModelId, onReselect) {
  // dismissible banner using the ADR-002 §Tier 1 #3 canonical copy:
  // "The model in your exported settings is no longer available. Please select a different model above."
  // + dropdown of STABLE_MODELS; onReselect(modelId) updates AppState and clears the banner
}
```

- **ADR-004 trigger (import time):** after parsing imported JSON, if `selectedModel` is not in `STABLE_MODELS`, call `showModelRecovery(imported.selectedModel, …)`.
- **ADR-002 trigger (runtime):** in `evaluateAnswer()`, on HTTP 400/404 for the selected model, catch the error, call `showModelRecovery(AppState.get('selectedModel'), …)`, and block further submissions until the user reselects.

Both share the same `showModelRecovery()` — not two UIs.

### 8. Rendering safety: sanitize all model HTML

Gemini's `error_analysis`, `polished_revision`, and `key_recommendations` are rendered as HTML via Marked.js. They are prompt-constrained but not trusted: a user can paste malicious content into the answer textarea that surfaces in the model's output. Every field is sanitized before injection:

```javascript
const html = DOMPurify.sanitize(marked.parse(markdown));
examinerOutput.innerHTML = html;
```

No `innerHTML` assignment from model output without `DOMPurify.sanitize()`. This pattern already exists in `index.html` and must be preserved through the restructure.

### 9. CDN fallback: unstyled but functional

"Offline-capable UI" (ADR-001) is defined as **all functionality works without CDN resources; styling degrades**.

- **Tailwind (CDN):** presentational only. If it fails, the page renders with browser-default styles — every control still works, just unstyled.
- **Marked.js (CDN):** required to render evaluation output. If missing, `evaluateAnswer()` detects the absent global, suppresses the API call (no wasted quota), and shows: *"Rendering libraries failed to load — evaluation output cannot be displayed. Check your connection."*
- **DOMPurify (CDN):** required for safe rendering. If missing, rendering is blocked entirely with the same message — safe default, never render unsanitized HTML.

**No vendored fallback.** Checking Tailwind (~500 KB), Marked (~30 KB), and DOMPurify (~10 KB) into the repo would bloat the deployment for a degradation path that is acceptable as-is. A ~20-line critical-CSS block for basic layout is a noted v2 enhancement.

### 10. State management: plain module-level store

A single `AppState` singleton (`scripts/store.js`) holds all cross-cutting state with `get()` / `set()` / `subscribe()`:

```javascript
// scripts/store.js
export const AppState = {
  data: {
    apiKey: '', selectedModel: '', selectedPart: 1, selectedQuestionId: '',
    currentAnswer: '', answerHistory: [], settings: { autoSave: true, highConfidenceMode: false },
    questions: [], capabilities: {}, // capabilities: derived model flags (e.g. temperature support)
  },
  get(k) { return this.data[k]; },
  set(k, v) { this.data[k] = v; this._subs.forEach(cb => cb(k, v)); },
  subscribe(cb) { this._subs.push(cb); },
  save() { /* localStorage, try/catch, per ADR-001 */ },
  export() { /* JSON, per ADR-001/ADR-004 */ },
  import(json) { /* validates questions, checks model, per ADR-004 */ },
};
```

No Redux/Zustand/jotai — a singleton with `subscribe()` is sufficient for state shared across 5–10 components. Components subscribe and re-render on the slices they care about; `AppState` is the single owner of selection, model, answer, history, settings, and the capability map.

### 11. General API error handling: one shared pattern

All API failures route through one `handleApiError(error, response)` in `scripts/evaluator.js`:

```javascript
// scripts/evaluator.js
export function handleApiError(error, response) {
  if (response?.status === 429) return 'Rate limit exceeded. Wait a moment and try again.';
  if (response?.status === 400 || response?.status === 404) {
    showModelRecovery(AppState.get('selectedModel')); // ADR-002 dead-model path (shared with §7)
    return null; // suppresses the generic message
  }
  if (response?.status === 401 || response?.status === 403) return 'Invalid API key. Please re-enter your Gemini API key.';
  if (error?.name === 'TypeError') return 'Network error. Check your connection and try again.';
  return 'Evaluation failed. Please try again.'; // generic — never a raw API dump
}
```

Each class has one actionable, user-facing message; raw API error text is never surfaced. The 400/404 model-deprecation path delegates to the shared `ModelRecoveryBanner` rather than spawning a new UI.

## Alternatives Considered

- **A1. Alpine.js (CDN, no build).** Rejected: reactivity is unnecessary at this scale; kept as the recommended upgrade path if the component count grows. Does not violate ADR-001's spirit.
- **A2. React + Vite (dev-only build, committed output).** Rejected: would require amending ADR-001's no-build constraint; adds an npm toolchain; ~40 KB for an app with a small question bank.
- **A3. Preact + htm (CDN).** Rejected: still a framework dependency; tagged-template syntax is unfamiliar to curators; no advantage over vanilla at this scale.
- **A4. Lit (Web Components).** Rejected: ~15 KB; the custom-element lifecycle is more ceremony than `render()` / `update()` / `destroy()`.
- **A5. zod / ajv runtime validation.** Rejected: adds a dependency and/or build step; the hand-rolled validator is already written and matches the no-build constraint.
- **A6. Redux / Zustand / jotai for state.** Rejected: bundle size and cognitive overhead unjustified for 5–10 components; the subscribe singleton suffices.

## Consequences

### Positive

- **Zero ADR-001 violations.** No build pipeline, single static HTML, GitHub Pages, BYOK, offline-capable — all preserved.
- **Model config delegated, not duplicated.** `STABLE_MODELS` / `TEMPERATURE_SUPPORT` live in `model-provider-guide.md` (source of truth) and `scripts/models.js` (mirror); this ADR references them, avoiding the staleness of an embedded list.
- **Framework-portable contracts.** The `render()` / `update()` / `destroy()` pattern maps to Alpine/Lit/React if a future ADR adopts one.
- **Centralized error + recovery.** One `handleApiError()` and one `ModelRecoveryBanner` serve every call site and both ADR-002/ADR-004 triggers.
- **Safe rendering, deliberate state.** DOMPurify on all model HTML; one explicit `AppState` shape instead of ad-hoc globals.

### Negative

- **Manual model-list sync.** `models.js` must be re-synced from the guide before each release. Intentional (per ADR-002 Tier 1 #3) and already a checklist item.
- **Manual re-render.** Without a reactivity framework, components must explicitly re-render on state change via `subscribe()`. Manageable at this scale; painful beyond ~50 components.
- **TypeScript types are documentation-only.** Runtime enforcement relies on the hand-rolled validator staying in sync with `question-schema.md`.
- **Unstyled degradation.** If the Tailwind CDN fails, the UI works but looks bare until a v2 critical-CSS block lands.

### Neutral

- **Multi-file load.** More HTTP requests than one `index.html`, irrelevant at this scale under HTTP/2.
- **Phase 1→2 is transparent.** Both phases return `Question[]`; `QuestionSelector` is unaffected by the migration.
- **Offline = unstyled-but-functional.** A deliberate tradeoff; full offline rendering is a noted v2 enhancement.

## Cross-References

- **ADR-001** — honors the no-build / single-static-HTML / GitHub-Pages / BYOK / offline constraints; resolves the "build pipelines" out-of-scope item with an explicit no-build decision.
- **ADR-002** — §Tier 1 #3 (shared dead-model recovery banner, §7), §Tier 1 #4 (`prompt_version` provenance, §5), §Tier 2 #1 (few-shot anchors, §5), and the conditional-temperature determinism strategy (§6).
- **ADR-004** — import-time dead-model check (§7 call site 1) and the Phase 1/2 question storage + `loadQuestions()` pseudocode that §3 implements.
- **`docs/question-schema.md`** — the TypeScript discriminated union and `validateQuestion()` pseudocode this ADR ports to runtime (§4); source of `RUBRIC_CRITERIA` / `RubricCriteriaPart1/2/3` (§5).
- **`docs/model-provider-guide.md`** — source of truth for `STABLE_MODELS` and `TEMPERATURE_SUPPORT`, mirrored into `scripts/models.js` (§6).
