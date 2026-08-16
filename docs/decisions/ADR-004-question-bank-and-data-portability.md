# ADR-004: Question Bank Strategy and Data Portability

## Status

Accepted

## Date

2026-08-16

## Context

ADR-001 requires a scalable question bank grouped by TOEIC Part (1, 2, 3) with a hierarchical Part → Question selector, plus JSON export/import of user state. It also requires an offline-capable UI and a single static HTML file deploy to GitHub Pages.

TOEIC Writing is a trademark of ETS. ETS licensing policy prohibits embedding actual TOEIC test questions for test-preparation purposes without explicit permission. The app must therefore use **original prompts** that follow the TOEIC format without reproducing copyrighted material.

ADR-002 requires the Gemini API response to conform to a structured JSON schema (criterion decomposition, provenance tags). The question bank must align its `rubricCriteria` field to the official ETS sub-criteria per Part (documented in ADR-002 Context and verified against the ETS Score User Guide).

model-provider-guide.md defines the Gemini model lineup and deprecation schedule. The question bank is independent of model choice but the export/import JSON may carry a `selectedModel` field that must be validated against the guide's stable model list on import (see "Dead-Model Recovery" below).

### ETS Official Question Format (Internal Reference Only)

The ETS TOEIC Writing sample test (ets.org, publicly available PDF) defines the canonical format for each Part. These are **not embedded in the app** — they are used only as style templates for human curation:

- **Part 1**: "Write ONE sentence based on a picture. Use the TWO words or phrases provided." (Score 0–3: grammar, relevance)
- **Part 2**: "Respond to a written request by email. Your response will be scored on quality and variety of sentences, vocabulary, and organization." (Score 0–4)
- **Part 3**: "Write an essay stating, explaining, and supporting your opinion on an issue. Minimum 300 words." (Score 0–5: supporting opinion, grammar, vocabulary, organization)

## Decision

### Storage

- **Phase 1**: Embed the question bank as a JSON array inside an inline `<script type="application/json">` tag in `index.html`. A `loadQuestions()` function (see `docs/technical-notes/file-structure-and-load-order.md`) parses the embedded JSON synchronously — no `fetch()` required. The data is available immediately after HTML parse, supporting `file://` protocol and offline use. Total payload is small — a few KB for the initial bank — and loads synchronously with no `fetch()`.
- **Phase 2** (when the bank exceeds ~30 questions or 50 KB): Split into a separate `questions.json` file served from the same GitHub Pages origin, loaded via `fetch()` with a localStorage cache fallback:

  ```javascript
  // Pseudocode — see ADR-003 for integration
  // Phase 1: loadQuestions() is synchronous (parse embedded JSON).
  // Phase 2: loadQuestions() becomes async (fetch + cache fallback).
  async function loadQuestions() {
    try {
      const res = await fetch('questions.json', { cache: 'no-cache' });
      const data = await res.json();
      localStorage.setItem('questions_cache', JSON.stringify(data));
      return data;
    } catch {
      const cached = localStorage.getItem('questions_cache');
      return cached ? JSON.parse(cached) : FALLBACK_QUESTIONS;
    }
  }
  ```

  This preserves offline UI capability: if GitHub Pages is up, the fetch succeeds; if the page is opened via `file://`, the cache or fallback is used.

### Generation

- **Phase 1**: Questions are **human-curated originals**. A human writer uses the ETS sample format as a template and creates new prompts with different scenarios, word pairs, and essay topics. The `copy-editing` skill (in `.pi/skills/`) is used to apply Clarity, Voice & Tone, and Specificity sweeps to ensure questions sound natural and not AI-generated.
- **Phase 2** (optional): AI-assisted generation becomes available as an authoring tool. The writer runs a calibrated Gemini prompt that generates candidate questions and self-critiques each for naturalness (1–10 rating). The writer reviews 3–5 generated questions before accepting the batch. **This is an authoring-time tool, not a runtime feature** — questions are frozen in the JSON before deploy, never generated at runtime.

### Quality Gate

Every question must pass four checks before entering the bank:

1. **Style alignment**: Matches the ETS TOEIC Writing format (not a generic English test).
2. **Topic/vocabulary check**: Author references `docs/reference-library.md` for workplace scenarios and business vocabulary authenticity (COCA business register, ABEL word list).
3. **Copy-editing sweep**: The `copy-editing` skill reviews Clarity (is the task unambiguous?), Voice & Tone (appropriate business register?), and Specificity (concrete enough to be engaging?).
4. **Human sniff-test**: At least one human writer confirms the question "sounds like it could be on the real TOEIC."

### Export / Import

The JSON export/import (ADR-001 scope) covers **user-generated data only**:

| Field | Export? | Notes |
| --- | --- | --- |
| `apiKey` | ✅ (optional checkbox) | User explicitly opted in; flagged as sensitive in UI |
| `selectedModel` | ✅ | Validated against model-provider-guide stable list on import |
| `selectedPart` / `selectedQuestionId` | ✅ | Cross-referenced against current question bank on import |
| `currentAnswer` | ✅ | Optional — user may want to carry over draft text |
| `answerHistory` | ✅ | Full history with scores and timestamps |
| `settings` | ✅ | Preferences (auto-save toggle, etc.) |
| Question bank | ❌ | Immutable per release — not part of user export |

**Schema versioning**: Export JSON includes `"version": 1`. On import, if `version` is lower than the current app version, a migration function upgrades the data structure (additive only — never removes fields).

**Dead-model recovery**: On import, if `selectedModel` matches a deprecated model ID in model-provider-guide.md's deprecation table, the app shows a banner using the canonical recovery copy from ADR-002 §Tier 1 #3 — "The model in your exported settings is no longer available. Please select a different model above." — and prompts the user to choose from the current stable list.

> **Cross-reference**: ADR-002 §Tier 1 #3 defines a complementary **runtime** recovery path — if `evaluateAnswer()` returns an API 400/404 for a saved model ID at evaluation time, a banner prompts reselection. The two checks are orthogonal: ADR-004's import-time check is proactive (prevents evaluation from failing at all); ADR-002's runtime check is reactive (handles models that become deprecated between visits). Both share the same banner UI and reselection flow — do not implement as two separate code paths.

### Schema

The canonical question schema is documented in `docs/question-schema.md`. Each question carries its `rubricCriteria` aligned to ETS official sub-criteria per Part:

- Part 1 (0–3): `grammar`, `task_completion`, `word_usage`
- Part 2 (0–4): `sentence_quality_variety`, `vocabulary`, `organization`
- Part 3 (0–5): `thesis_and_support`, `grammar`, `vocabulary`, `organization`

In addition, Part 2 questions carry an optional `directives` array describing what the response must contain (typically two reasons plus one question, per the ETS Part 2 format), and Part 3 questions carry an optional `title` for display (mirroring an email subject). Both are additive, optional fields; see `docs/question-schema.md` and `docs/toeic-format-reference.md`. These content-coverage fields are consumed by the ADR-002 evaluator: `buildQuestionText()` injects them into the examiner prompt so the AI examiner checks task completion alongside rubric criteria.

## Alternatives Considered

### External API-backed question bank

- **Pros**: Unlimited questions, real-time updates
- **Cons**: Violates the "no backend" constraint; introduces a dependency; breaks offline UI
- **Rejected**: ADR-001 explicitly requires static-only deployment

### IndexedDB at runtime

- **Pros**: Queryable, can store large datasets, user-custom questions
- **Cons**: Adds complexity for a dataset that is only a few KB; no benefit at Phase 1 scale; IndexedDB quirks across browsers
- **Rejected**: Overkill for <50 questions; deferred to future ADR if user-custom questions are added

### sql.js-httpvfs (SQLite on GitHub Pages)

- **Pros**: Full SQL querying on static host, handles large datasets, B-tree indexes
- **Cons**: ~1 MB WASM binary; unnecessary for a flat question array; adds startup latency
- **Rejected**: Extreme overkill for the dataset size; noted as a future option only if the bank grows beyond 1000 questions with complex relationships

### Headless CMS (Contentful, Sanity, etc.)

- **Pros**: Content editing UI, team collaboration
- **Cons**: External service dependency; API key in frontend (conflicts with BYOK purity); monthly cost; breaks offline-first
- **Rejected**: Violates the client-side-only and offline-capable constraints

### Embed actual ETS questions

- **Pros**: Authentic difficulty, guaranteed format alignment
- **Cons**: Copyright violation per ETS licensing policy
- **Rejected**: Explicitly prohibited

### Internal Reference File: `docs/questions-sample.json`

The ETS sample questions and additional reference samples (TestSUCCEED) are stored in `docs/questions-sample.json` for **internal calibration only**. This file is listed in `.gitignore` and is **not part of the deployed static site** — it is not committed to the repo and therefore not served by GitHub Pages. Curators must distribute it separately (e.g., via private email) to anyone who needs it. The file must never be referenced by `index.html` or any runtime code path.

If this file is accidentally committed to the public repo served by GitHub Pages, it is exposed at a predictable URL. The `.gitignore` entry is the primary safeguard — no runtime code path should ever reference or `fetch()` this file.

Structure: the file is a JSON object with a `_notice` field (copyright safety notice — do not embed or publish) and a `questions` array. The `_notice` field must be preserved and updated whenever sources change.

Note: calibration file question IDs use a non-conforming convention (`ets-p1-01`, `ts-p2-01`) and are intentionally excluded from `validateQuestion()`. Real authored questions must follow the `p${part}-` prefix convention.

## Consequences

### Positive

- **Copyright-safe**: Original questions only — no licensing risk
- **Offline-first**: Phase 1 embedded JSON works with zero dependencies, even in `file://` mode
- **Version traceability**: Question bank is committed to the repo, versioned alongside the app — every deploy has a fixed set of questions
- **Quality control**: Human curation + copy-editing sweeps + sniff-test ensure natural, authentic-sounding prompts
- **Migration path**: Clear upgrade story from embedded JSON → separate file → (if needed) IndexedDB or CMS

### Negative

- **Human bottleneck**: Questions must be hand-curated, slowing bank expansion
- **Immutability friction**: Adding questions requires a new deploy (HTML edit + commit + GH Pages rebuild)
- **No user-generated questions**: Users can't add their own questions to the bank (only export their answer history)

### Neutral

- **Question bank is immutable per release**: This is intentional — questions are part of the app, not user data
- **Phase 2 migration is non-breaking**: Moving from inline JSON to a separate `questions.json` file is transparent to users
- **Export/import versioning**: Version 1 schema is the starting point; future versions are additive

## Cross-References

- **ADR-001**: Defines the scope and constraints (static-only, BYOK, offline-capable, JSON export/import)
- **ADR-002**: Defines the rubric criteria that each question's `rubricCriteria` field must align to
- **ADR-003**: Documents the component that consumes this schema (`QuestionSelector`, `loadQuestions()` integration)
- **model-provider-guide.md**: Defines the stable model list used for validating imported `selectedModel` fields
- **docs/question-schema.md**: Canonical schema reference
- **`.pi/skills/copy-editing/`**: Question quality review tool (Clarity, Voice & Tone, Specificity sweeps)
