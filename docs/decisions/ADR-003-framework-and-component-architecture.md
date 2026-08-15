# ADR-003: Framework and Component Architecture for the Static-Only App

## Status

Proposed

## Date

2026-08-15

## Context

ADR-001 defines the app as a single-page, client-side TOEIC Writing practice tool with hard constraints: **single static HTML file, no backend, no build pipelines, GitHub Pages deployment, BYOK, and offline-capable UI**. The current `index.html` is 380 lines of inline vanilla JavaScript with no external dependencies beyond CDN-loaded Tailwind, Marked.js, and (recently added) DOMPurify.

The app is being restructured from a flat inline script into a component-level architecture for extensibility. This requires resolving several interdependent decisions that prior ADRs have explicitly deferred to this document:

1. **Framework choice vs. the no-build-pipeline constraint.** ADR-001 rules out build pipelines. A framework like React, Vue SFCs, or SvelteKit requires a dev-time build step to produce the final static HTML. Whether that's compatible (dev-only build, committed static output) or not needs an explicit decision.
2. **Component decomposition.** ADR-001 and ADR-004 reference a `QuestionSelector` component and `loadQuestions()` integration — both need defined data contracts.
3. **`validateQuestion()` execution.** `docs/question-schema.md` defines a TypeScript discriminated union and a `validateQuestion()` pseudocode function, but TypeScript types do not validate JSON at runtime. The mechanism (hand-rolled JS validator vs. schema validator vs. build-time generation) needs a concrete resolution.
4. **Prompt assembly.** ADR-002 defines per-Part rubric criteria (`RubricCriteriaPart1/2/3` from `question-schema.md`) and few-shot anchors. The `SYSTEM_PROMPT` constant in `index.html` must be restructured to inject per-Part criteria dynamically.
5. **Model dropdown.** `model-provider-guide.md` defines the stable model list and temperature-support map. The dropdown must use a baked-in list, not a live `models.list()` API call (which would violate offline capability).
6. **Dead-model recovery.** ADR-002 (runtime 4xx on `evaluateAnswer()`) and ADR-004 (import-time check against the deprecation table) both point to one shared implementation.
7. **Rendering safety.** Gemini's `error_analysis`, `polished_revision`, and `key_recommendations` text fields are rendered as HTML via Marked.js. These come from a prompt-driven LLM response and are not fully trusted.
8. **CDN fallback.** ADR-001 requires the UI to work when CDN resources are unavailable. The current app loads Tailwind, Marked, and DOMPurify from CDN — "works" needs a concrete definition.
9. **State management shape.** Cross-cutting state (selected model, Part/question, draft answer, history, API key, settings, capability map) needs a deliberate structure rather than ad-hoc globals.
10. **General API error handling.** Beyond dead-model recovery, the app must handle rate limits, quota errors, malformed responses, and timeouts with a shared pattern.

## Decision

### 1. Framework: Stay vanilla JS — no framework, no build step

**Decision**: Retain **vanilla JavaScript with ES modules** as the sole framework. Do not adopt Alpine.js, React, Preact, Lit, Vue, or any other framework. Accept a dev-only build step **only if** it produces committed static output that is deployed as-is (i.e., the build is a curator-time convenience, not a runtime dependency). As of this ADR, no build step is added.

**Rationale** (in priority order):

- **ADR-001 compliance**: ADR-001 explicitly rules out "build pipelines." A framework that requires bundling to produce the deployed HTML is a non-starter without a conscious amendment to that constraint. Even a dev-only build introduces complexity (dependency management, CI, lockfile churn) for a dataset that fits in under 2 KB.
- **Constraint stack**: GitHub Pages (static-only), no backend, BYOK, offline-capable UI — vanilla JS satisfies all four with zero tradeoffs. Every framework considered adds at least one compromise.
- **Current trajectory**: The existing `index.html` is already vanilla JS. The user explicitly prefers "a lighter framework than React." The lightest option is *no* framework — the "component-level organization" the user wants can be achieved through module pattern, which is strictly additive to what already works.
- **Bundle size**: Zero bytes of framework overhead. Tailmarked Alpine.js is ~10 KB gzipped; React + deps is ~40 KB+. Not critical at this scale, but the principle holds: every dependency is a failure surface.
- **Extensibility debt is cheap**: A well-organized vanilla module structure can be progressively enhanced. If the app grows beyond the current scope, the framework question can be revisited in a new ADR — but the migration path from vanilla modules to any framework is far easier than the reverse.

**Component-level organization**: The "components" are implemented as **self-contained module functions** following a `createComponent(props, ctx)` convention:

```javascript
// Example: a Component function returns { render(), update(props), destroy() }
function QuestionSelector({ questions, state, onQuestionChange }) {
  return {
    render() { /* return HTML string */ },
    update(newProps) { /* diff and patch DOM */ },
    destroy() { /* remove event listeners */ }
  };
}
```

Each component is a single function with a `render`/`update`/`destroy` lifecycle, stored in its own `.js` file. State flows in one direction: `AppState` → `props` → `Component.render()`. This is a deliberate, minimal emulation of the framework component contract — enough structure to decompose features, not enough to become a mini-framework.

> **Future option**: If component reactivity becomes a genuine need (not just a preference), **Alpine.js 3.x** loaded from CDN is the lowest-friction upgrade path — it's CDN-loadable (no build step), ~10 KB gzipped, and maps naturally from the existing x-data-style thinking. This is the only framework on the shortlist that doesn't violate ADR-001's spirit.

### 2. File structure: Multi-file `<script type="module">` on a single HTML page

The app remains a **single static HTML page** deployed via GitHub Pages. The inline `<script>` block is restructured into multiple `<script type="module">` files referenced via `src=`:

```
/index.html                    — HTML skeleton + component wiring
/scripts/store.js              — singleton AppState
/scripts/schema.js             — QuestionType constants + validateQuestion()
/scripts/questions.js          — embedded question bank (Phase 1 inline JSON array)
/scripts/components/
  /modelDropdown.js            — ModelDropdown component
  /questionSelector.js         — QuestionSelector component
  /answerEditor.js             — AnswerEditor component
  /modelRecoveryBanner.js      — shared dead-model recovery banner
  /examinerOutput.js           — evaluation results renderer
/scripts/promptBuilder.js      — assembles SYSTEM_PROMPT per Part
/scripts/evaluator.js          — evaluateAnswer() + error handling
/scripts/app.js                — top-level controller, mounts components
```

**Why not separate HTML pages or an SSG?** One page = one deploy artifact. The app has no need for routable pages beyond Part → Question selection, which is in-page state.

### 3. validateQuestion() mechanism: Hand-rolled JS validator

`docs/question-schema.md` defines TypeScript-level types (discriminated union, literal unions for criteria names), but **types do not run at runtime**. The actual validator is a hand-rolled JavaScript function that mirrors the pseudocode in `question-schema.md`, ported faithfully:

```javascript
// In scripts/schema.js — no build step, no zod/ajv dependency
function validateQuestion(q) {
  assert(q.part === 1 || q.part === 2 || q.part === 3);
  assert(q.id.startsWith(`p${q.part}-`));
  assert(q.scoringBand.min === 0);
  assert(q.scoringBand.max === { 1: 3, 2: 4, 3: 5 }[q.part]);
  assert(q.rubricCriteria.length > 0);
  if (q.part === 1) assert(q.targetWords.length === 2);
  if (q.part === 2) assert(q.emailContext != null);
  if (q.part === 3) assert(q.essayTopic && q.essayTopic.length > 10);
  return true;
}
```

**Call sites**:

- **Init time**: `validateQuestion()` runs on every question in the embedded bank (Phase 1 `questions.js`, or `questions.json` in Phase 2). Failing assertions throw before the UI renders — this is a dev-time bug, not a user-facing error.
- **Import time**: When the user imports JSON, each question in their `questionBank` export is validated. Malformed questions are rejected with a user-facing message (not a crash), since user data crosses trust boundaries.

**Why not zod/ajv?** Adds a dependency and a build step for runtime schema validation — unnecessary when the validator is already written and the schema is stable (additive-only versioning per ADR-004).

**Why not TypeScript compilation?** Would require a build step (tsc) to strip types and emit `.js`. The TypeScript interface and literal unions in `question-schema.md` serve as the type-level source of truth for future tooling, but the runtime validator is plain JS. This is a deliberate decoupling: the schema doc is the design artifact, the JS validator is the runtime enforcement.

### 4. QuestionSelector + loadQuestions integration

**Data contract**: `QuestionSelector` receives `{ questions: Question[], value: { part, questionId }, onChange }`. It renders a two-level selector:

- **Level 1**: Dropdown or button group for Part (1, 2, 3). Filtering is by `q.part`.
- **Level 2**: Dropdown of questions within the selected Part. Filtering is by `q.id.startsWith('p${part}-')`.

**Loading strategy**:

- **Phase 1** (up to ~30 questions): `questions.js` exports a `QUESTIONS` array — inlined via `<script type="module">`. `loadQuestions()` is a trivial async wrapper that resolves immediately:

```javascript
// scripts/questions.js
export const QUESTIONS = [/** ... embedded array ... */];

export async function loadQuestions() {
  return QUESTIONS; // resolves synchronously in practice
}
```

- **Phase 2** (separate `questions.json` file): `loadQuestions()` does `fetch('questions.json')` with a `localStorage` cache fallback and a `FALLBACK_QUESTIONS` constant for `file://` or network failure. The `catch` block parses `localStorage.getItem('questions_cache')`; if that fails too, it falls back to the built-in `FALLBACK_QUESTIONS` array (same as Phase 1).

**`QuestionSelector` does not call `loadQuestions()`** itself — data loading is lifted to `AppState` in `app.js`, which calls `loadQuestions()` once at init and stores the result. Components receive the data via props, making the loading strategy transparent.

### 5. SYSTEM_PROMPT and prompt assembly

The monolithic `SYSTEM_PROMPT` constant is replaced by a **`PromptBuilder`** module (`scripts/promptBuilder.js`) that assembles the system prompt dynamically:

```javascript
// scripts/promptBuilder.js
import { RUBRIC_CRITERIA } from './schema.js';
import { FEW_SHOT_ANCHORS } from './fewShotAnchors.js';

function buildSystemPrompt(part) {
  const criteria = RUBRIC_CRITERIA[part]; // e.g. ['grammar', 'task_completion', 'word_usage']
  const anchors = FEW_SHOT_ANCHORS[part];
  return `
You are an expert ETS TOEIC Writing examiner. Evaluate strictly against ETS criteria.

Scoring band for this Part: ${SCORING_BANDS[part]}
Rubric criteria to decompose: ${criteria.join(', ')}
...
Few-shot examples:
${anchors}
  `;
}

function buildEvaluationPrompt(question, answer) {
  return `
Question Prompt:
${question.prompt}
${question.emailContext ? `Email: ${JSON.stringify(question.emailContext)}\n` : ''}
${question.essayTopic ? `Essay Topic: ${question.essayTopic}\n` : ''}
Target words: ${question.targetWords?.join(', ') || 'N/A'}
${question.wordTarget ? `Word target: ${question.wordTarget.min}–${question.wordTarget.max} words\n` : ''}

Student Answer:
${answer}
  `;
}
```

**Key decisions**:

- Criteria names come from `RubricCriteriaPart1/2/3` in `question-schema.md` — imported as constants from `schema.js`, never hardcoded in strings.
- Few-shot anchors (ADR-002 Tier 2 #1) are stored in a separate `fewShotAnchors.js` module — one pair per Part, paraphrased for copyright safety.
- `prompt_version` is a constant in `PromptBuilder` — bumped whenever the system prompt changes. Returned in the evaluation response `provenance` fields per ADR-002 §Tier 1 #4.

### 6. Model dropdown: baked-in static list

The dropdown is populated from a **hardcoded** `STABLE_MODELS` array and a `TEMPERATURE_SUPPORT` map in `scripts/modelDropdown.js`, sourced from `model-provider-guide.md` at implementation time. No `models.list()` API call is made at runtime.

```javascript
// scripts/modelDropdown.js
const STABLE_MODELS = [
  { id: 'gemini-3.6-flash', label: 'Gemini 3.6 Flash', temperatureSupported: false },
  { id: 'gemini-3.5-flash', label: 'Gemini 3.5 Flash', temperatureSupported: false },
  { id: 'gemini-3.5-flash-lite', label: 'Gemini 3.5 Flash-Lite', temperatureSupported: false },
];

function getTemperature(modelId) {
  const model = STABLE_MODELS.find(m => m.id === modelId);
  return model?.temperatureSupported ? 0.2 : undefined; // undefined = omit from generationConfig
}
```

**Update rule**: When `model-provider-guide.md` is updated (before each release per ADR-002), the `STABLE_MODELS` array in this file must be re-synced manually. This is an explicit checklist item in the release process (linked to ADR-002 Tier 1 #3).

**Live discovery**: Not implemented. ADR-001 §Constraints requires offline-capable UI. A live `models.list()` call would fail offline and silently empty the dropdown. If the user wants a "refresh model list" button in v2, that is an enhancement, not a dependency.

### 7. Dead-model recovery: one banner, two triggers

A single `ModelRecoveryBanner` component (`scripts/components/modelRecoveryBanner.js`) renders a dismissible alert using the canonical recovery copy from ADR-002 §Tier 1 #3 — *"The model in your exported settings is no longer available. Please select a different model above."* — with a dropdown of `STABLE_MODELS`.

It is triggered from **two call sites** (per ADR-002 and ADR-004), not implemented as two separate UIs:

- **ADR-004 trigger (import-time)**: After parsing imported JSON, if `selectedModel` is not in `STABLE_MODELS`, show the banner pre-filled with the deprecated model ID.
- **ADR-002 trigger (runtime)**: In `evaluateAnswer()`, if the provider returns HTTP 400/404 for the selected model, catch the error, show the banner, and block further submissions until the user reselects.

Both call sites share the same `showModelRecovery(deprecatedModelId)` function, which mounts the banner and returns the user's reselection via a callback.

### 8. Rendering safety: DOMPurify (already integrated)

All markdown rendered from Gemini's response is sanitized with **DOMPurify** (already loaded via CDN in `index.html`):

```javascript
const html = DOMPurify.sanitize(marked.parse(markdownText));
analysisResult.innerHTML = html;
```

**Non-negotiable**: No `innerHTML` assignment from LLM output without `DOMPurify.sanitize()`. The existing `index.html` already does this correctly; the pattern must be preserved in the restructured code.

**Error response fields** (`error_analysis`, `polished_revision`, `key_recommendations`): These come from a prompt-constrained LLM but could still contain prompt-injection payloads if the user pastes malicious content into the textarea. DOMPurify is the safety layer; it strips `<script>`, event handlers, and other XSS vectors.

### 9. CDN fallback: unstyled but functional

**"Offline-capable UI"** (ADR-001 §Driving Requirements #5) is defined as: **all functionality works without CDN resources, but the UI is unstyled**.

- **Tailwind (CDN)**: purely presentational. If the CDN fails, the page renders with browser default styles — all buttons, inputs, and text are still functional. No layout, but no broken functionality.
- **Marked.js (CDN)**: Required for rendering evaluation output. If the CDN fails, `evaluateAnswer()` detects the missing `marked` global and displays: *"Marked.js failed to load — evaluation output cannot be rendered. Check your network connection."* The evaluation API call is suppressed (no wasted quota).
- **DOMPurify (CDN)**: Required for safe rendering. If the CDN fails, rendering is blocked entirely with the same message above. Safe defaults — never render unsanitized HTML.

**Why not vendor local fallbacks?** Three library copies (Tailwind ~500 KB, Marked ~30 KB, DOMPurify ~10 KB) checked into the repo would bloat the deployment. The presentational degradation (no Tailwind) is acceptable; the functional degradation (no Marked/DOMPurify) is blocked with a clear message. If the user wants full offline rendering in v2, this is an explicit tradeoff to revisit.

**Future option**: A minimal `<style>` block with ~20 lines of critical CSS could provide basic layout without Tailwind. Noted but not implemented in v1.

### 10. State management: singleton `AppState`

A single `AppState` singleton (`scripts/store.js`) holds all cross-cutting state with a `get()`/`set()`/`subscribe()` API:

```javascript
const AppState = {
  data: {
    apiKey: '',
    selectedModel: 'gemini-3.6-flash',
    selectedPart: 1,
    selectedQuestionId: 'p1-001',
    currentAnswer: '',
    answerHistory: [],
    settings: { autoSave: true, highConfidenceMode: false },
    questions: [],
    stableModels: STABLE_MODELS,
  },
  get(key) { return this.data[key]; },
  set(key, value) {
    this.data[key] = value;
    this._subscribers.forEach(cb => cb(key, value));
  },
  subscribe(cb) { this._subscribers.push(cb); },
  // Persistence helpers
  save() { /* localStorage with try-catch, per ADR-001 */ },
  export() { /* JSON per ADR-001/ADR-004 */ },
  import(json) { /* validates questions, checks model, per ADR-004 */ },
};
```

**Why not Redux/Zustand/jotai?** All add bundle size and cognitive overhead. A singleton with subscribe is sufficient for state that lives across 5–10 components. If the app grows beyond ~2000 lines of component code, revisit in a new ADR.

**Why not `useStore` hooks?** No framework, no React. The subscribe pattern is the vanilla equivalent.

### 11. General API error handling

A shared `handleApiError(error, response)` function in `scripts/evaluator.js` classifies and surfaces errors:

```javascript
function handleApiError(error, response) {
  if (response?.status === 429) {
    return 'Rate limit exceeded. Please wait a moment and try again.';
  }
  if (response?.status === 400 || response?.status === 404) {
    const msg = error.message;
    if (msg.includes('model') || msg.includes('not found')) {
      // Dead-model recovery path (shared with ADR-004)
      showModelRecovery(AppState.get('selectedModel'));
      return null; // suppresses generic error message
    }
    return `API error: ${msg}`;
  }
  if (response?.status === 401 || response?.status === 403) {
    return 'Invalid API key. Please check and re-enter your Gemini API key.';
  }
  if (error.name === 'TypeError' && error.message.includes('fetch')) {
    return 'Network error. Check your connection and try again.';
  }
  return error.message;
}
```

**Key principle**: Error messages are user-facing strings, never raw API error dumps (which could leak internal details). Each error class has a specific, actionable message. The 400/404 model-deprecation path delegates to the shared `ModelRecoveryBanner` rather than creating a new UI.

## Alternatives Considered

### A1. Alpine.js (CDN-loadable, no build step)

- **Pros**: Component-level reactivity (`x-data`), CDN-loadable, ~10 KB gzipped, maps naturally from imperative DOM thinking
- **Cons**: Adds a dependency; different paradigm from existing `index.html`; the reactivity model is unnecessary for an app this small (5–10 components, infrequent UI updates)
- **Verdict**: Kept as the **recommended upgrade path** if the app outgrows vanilla modules. Not adopted now because the component count is low and the existing code already works.

### A2. React + Vite (dev-only build, committed static output)

- **Pros**: Mature ecosystem, extensive tooling, clear component mental model, user explicitly considered it
- **Cons**: Violates the spirit of ADR-001's "no build pipelines" constraint; requires `npm install` and a maintainer toolchain; bundle size (~40+ KB) for an app with <50 questions
- **Verdict**: **Rejected.** Only acceptable if ADR-001 is amended in a new ADR. The cost/benefit does not justify the complexity at this stage.

### A3. Preact + htm (CDN-loadable JSX alternative)

- **Pros**: CDN-loadable, ~10 KB gzipped, JSX-like syntax without a build step
- **Cons**: Still a framework dependency; `htm` tagged-template syntax is unfamiliar to curators; no clear advantage over vanilla modules at this scale
- **Verdict**: **Rejected.** Overkill for the current component count.

### A4. Lit (Web Components standard)

- **Pros**: Standards-based, framework-agnostic, good DX for custom elements
- **Cons**: Adds ~15 KB; custom element lifecycle (connectedCallback, observedAttributes) is more ceremony than the simple `render()/update()/destroy()` pattern used here
- **Verdict**: **Rejected.** Not enough component complexity to justify the overhead.

### A5. Zod/ajv runtime validation

- **Pros**: Declarative schema, auto-generated validators, strong type inference
- **Cons**: Adds a dependency; the `validateQuestion()` pseudocode is already written and simple enough to port to JS directly; schema changes are rare (additive-only versioning per ADR-004)
- **Verdict**: **Rejected.** Hand-rolled validator is simpler, dependency-free, and matches the no-build-step constraint.

## Consequences

### Positive

- **Zero constraint violations**: Stays fully within ADR-001's scope (no build pipeline, single static HTML, GitHub Pages, BYOK, offline-capable).
- **Low migration risk**: The existing `index.html` already works. The restructure is a pure refactor — same dependencies, same deployment, just better-organized code.
- **Type-safe at the design level**: The TypeScript discriminated union in `question-schema.md` serves as the type-level contract; the hand-rolled JS validator enforces the same rules at runtime. Two complementary layers.
- **Component contract is framework-portable**: The `createComponent(props, ctx) → { render(), update(), destroy() }` pattern maps trivially to Alpine.js, Lit, or React components if a future ADR adopts a framework.
- **Error handling is centralized**: One `handleApiError()` function for all API failures, with the dead-model recovery path shared between ADR-002 and ADR-004.

### Negative

- **Manual state management**: No reactivity framework means components must explicitly re-render when state changes. At 5–10 components this is manageable via `AppState.subscribe()`. At 50+ it would become painful.
- **No hot-reloading**: Curator edits to `.js` files require a browser refresh. Acceptable for a static site.
- **TypeScript types are documentation-only**: The discriminated union lives in the schema doc, not compiled. Runtime enforcement relies on the hand-rolled validator staying in sync.

### Neutral

- **File structure is explicit**: 10+ files in `/scripts/` is more files than a single `index.html`, but each has a clear purpose. The bundler-free approach means each file is loaded individually by the browser (more HTTP requests), but HTTP/2 multiplexing + small file sizes make this irrelevant at this scale.
- **Model list is hardcoded**: Requires manual sync with `model-provider-guide.md` before each release. This is intentional (see §6 Update rule) and is already a checklist item per ADR-002.
- **`questions.js` vs `questions.json`**: The Phase 2 migration from inlined array to `fetch()` is designed to be transparent to `QuestionSelector` — both paths produce a `Question[]` that `AppState` stores.
- **Offline rendering**: Tailwind-only fallback (unstyled but functional) is a deliberate tradeoff. A 20-line critical CSS block is a noted v2 enhancement.

## Cross-References

- **ADR-001**: Defines the no-build, single-static-HTML, GitHub-Pages, BYOK, offline-capable constraints this ADR respects. §3 (validateQuestion) and §10 (state management) directly implement ADR-001 scope items.
- **ADR-002**: §7 (error handling) and §7 (dead-model recovery) integrate with ADR-002's Tier 1 #3 (runtime 4xx recovery). §5 (prompt assembly) injects ADR-002's per-Part rubric criteria and few-shot anchors.
- **ADR-004**: §4 (loadQuestions + QuestionSelector) and §7 (dead-model recovery import-time check) implement ADR-004's documented pseudocode and recovery flow. §3 (validateQuestion) enforces the schema from `question-schema.md`.
- **`docs/question-schema.md`**: The TypeScript discriminated union + `validateQuestion()` pseudocode this ADR ports to JS.
- **`docs/model-provider-guide.md`**: Source of the `STABLE_MODELS` array and `TEMPERATURE_SUPPORT` map in §6.
