# Cross-Cutting Concerns

> Technical reference — implementation guidance for ADR-003 (cross-cutting concerns).

## State Management

A single `AppState` singleton on `App.store` holds all cross-cutting state:

`apiKey`, `selectedModel`, `selectedPart`, `selectedQuestionId`, `currentAnswer`, `answerHistory`, `settings`, `questions`, `capabilities`.

Internal state lives in the closure, not on the namespace object — only the `AppState` handle is public, preventing external mutation of internal state. Exposed via `get()` / `set()` / `subscribe()`.

## Rendering Safety

Gemini's evaluation output (error analysis, polished revision, key recommendations) is rendered as HTML via Marked.js. Every field is sanitized with DOMPurify before `innerHTML` assignment — never raw model output.

## CDN Fallback

"Offline-capable UI" (ADR-001) means all functionality works without CDN resources; styling degrades:

- **Tailwind:** presentational only. If the CDN fails, the page renders with browser defaults — every control works, just unstyled.
- **Marked.js:** required for evaluation output rendering. If missing, `evaluateAnswer()` detects the absent global, suppresses the API call (no wasted quota), and shows: "Rendering libraries failed to load — evaluation output cannot be displayed."
- **DOMPurify:** required for safe rendering. If missing, rendering is blocked entirely with the same message — safe default, never render unsanitized HTML.

No vendored fallbacks are checked in (Tailwind ~500 KB, Marked ~30 KB, DOMPurify ~10 KB). A critical-CSS block for basic layout is a noted v2 enhancement.

## Cross-reference

- **ADR-001:** Offline-capable UI (primary constraint)
- **ADR-002 Tier 1 #4:** "AI estimate" framing + provenance tags
- **ADR-003:** Component contract (see `component-contract.md`)
- **ADR-006:** `handleApiError()` shared error handler
