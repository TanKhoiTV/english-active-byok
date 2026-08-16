# Classic Scripts, Not ES Modules

> Technical reference — implementation rationale for ADR-003 (vanilla JS, no build step).

## Decision

Split into multiple classic `<script src>` files attached to a shared `window.App` namespace, loaded in explicit dependency order via `<script defer>` tags in `index.html`. Do **not** use `<script type="module">`.

## Alternatives Considered

### ES modules (`<script type="module">`) for the multi-file split

Rejected: module scripts are fetched under CORS rules and fail to load when `index.html` is opened via `file://`. The browser assigns an opaque `null` origin under `file://`, so the module fetch fails the CORS check before any JavaScript — including ADR-004's `loadQuestions()` fallback chain — ever executes. This breaks ADR-004's stated goal of supporting `file://` access for the embedded Phase 1 question bank.

## Rationale

The `file://` support requirement (ADR-004, Phase 1: "works with zero dependencies, even in `file://` mode"; Phase 2: "if the page is opened via `file://`, the cache or fallback is used") is the architectural keystone that rules out ES modules.

### The failure cascade

When `index.html` is opened via `file://`, the browser assigns an opaque `null` origin. ES module scripts (`<script type="module">`) are fetched as cross-origin requests relative to that `null` origin — the module fetch itself fails the CORS check before any JavaScript executes. This means the *entire script graph* never loads, not just `loadQuestions()`.

ADR-004's carefully designed fallback chain (`inline JSON → fetch → localStorage → FALLBACK_QUESTIONS`) never runs because the entry point module is blocked at the network layer. There is no fallback to the fallback.

### Why classic scripts work

Classic `<script src>` tags have no origin restriction — they load from the filesystem without any CORS check, identically under `file://` and `https://`. This is why the current `index.html` works when double-clicked from a file manager: every script (Tailwind, Marked, DOMPurify, inline app logic) loads as a classic script.

Classic scripts also unify the loading model: all existing CDN dependencies already load as classic globals, so the page would use two loading mechanisms (module + classic) if modules were introduced. One model is simpler and has fewer failure modes.

## Cost

Losing native `import`/`export`: each file wraps its contents in an IIFE and attaches only its public surface to `App.*`; internal helpers stay local to the closure instead of leaking into the shared global scope that all classic scripts execute in. Manual load-order management in `index.html` (see [File Structure and Load Order](file-structure-and-load-order.md)) replaces what `import` statements would normally provide.

## Escape hatch

A bundler configured to output a single classic `<script>` (not `type="module"`) would also satisfy the `file://` constraint. See [ADR-005]((../decisions/ADR-005-build-step-escape-hatch.md)) for the migration trigger conditions.

## Cross-reference

- **ADR-001:** Offline-capable UI constraint
- **ADR-003 D1:** Vanilla JS, no framework (primary decision)
- **ADR-003:** File-per-concern with explicit load order (see `file-structure-and-load-order.md`, depends on this)
- **ADR-004:** Phase 1 (inline JSON) and Phase 2 (`fetch` fallback) question storage
