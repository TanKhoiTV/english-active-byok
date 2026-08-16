# ADR-003: Framework and Component Architecture for the Static-Only App

## Status

Accepted

## Date

2026-08-16

## Context

The app is being restructured from a single monolithic `index.html` with an inline `SYSTEM_PROMPT` constant into a component-level architecture. ADR-001's "no build pipelines" constraint needs interpretation: does a dev-time build that emits committed static output count as satisfying "no build"?

ADR-001 defines the app as a single static HTML file deployed to GitHub Pages, with an offline-capable UI — the interface (question selection, answer writing, settings) must work even if CDN resources are unavailable. ADR-004 extends this beyond CDN-fallback to filesystem distribution: Phase 1 embeds the question bank inline so the page works "even in `file://` mode", and Phase 2's `loadQuestions()` fallback chain explicitly handles the case where the page is opened via `file://`.

The current `index.html` achieves this baseline with vanilla JS loading Tailwind, Marked.js, and DOMPurify from CDN. The restructure must preserve every constraint.

## Decision

Adopt vanilla JavaScript with a multi-file split into classic `<script defer>` files sharing a single `window.App` namespace. No framework, no build step, no ES modules. Components are plain functions with a minimal `render()` / `update(props)` / `destroy()` lifecycle contract, attached to `App.components`. State is managed by a single `AppState` singleton on `App.store`.

> ADR-006 documents the related decisions on error handling (shared handler + dead-model recovery) and runtime validation (hand-rolled validator), which are constrained by this decision's no-framework, no-build requirement.

## Alternatives Considered

### Dev-build framework (React + Vite, committed output)

- **Pros:** Component model, hot reload, ecosystem tools.
- **Cons:** Imports a toolchain, lockfile churn, and CI dependency for an app whose entire dataset is a small embedded question bank — a cost with no offset at this scale. Also violates ADR-001's "no build pipelines" out-of-scope item.
- **Rejected.**

### CDN framework (Alpine.js, Preact+htm, Lit)

- **Pros:** Avoids the build step while providing reactivity.
- **Cons:** Still adds a dependency and a reactivity model this app does not need (5–10 components, infrequent UI updates).
- **Rejected** for current scope. Kept as the recommended upgrade path if component count grows significantly — the `render()` / `update()` / `destroy()` pattern maps to any of these with a mechanical rename.

### Web Components (Lit)

- **Pros:** Standards-based, framework-agnostic.
- **Cons:** ~15 KB bundle size; the custom-element lifecycle is more ceremony than a `render()` / `update()` / `destroy()` contract.
- **Rejected.**

### ES module syntax with bundling to a single classic script

- **Pros:** Gains `import`/`export` syntax and automatic dependency resolution.
- **Cons:** Adds a build step, lockfile, and CI dependency. Import maps do not bypass CORS for `type="module"` fetches under `file://`, so module scripts still fail in local mode.
- **Rejected** for current scope. Kept as the upgrade path documented in [ADR-005](../decisions/ADR-005-build-step-escape-hatch.md).

## Rationale

ADR-001's "no build pipelines" is read as written. Vanilla scripts give component decomposition with zero framework overhead and the smallest failure surface. The component contract (`render`/`update`/`destroy`) is deliberately minimal so that adopting Alpine, Lit, or React later is a mechanical rename (drop the IIFE wrapper, add an `export`), not a rewrite.

ADR-001's constraint language permits a "static bundle" but its driving requirements prohibit "build pipelines" and its out-of-scope excludes "build pipelines." This ADR interprets that as permitting pre-committed static artifacts (e.g., a manually concatenated single JS file) but not deploy-time build steps. This interpretation is revisited in [ADR-005](../decisions/ADR-005-build-step-escape-hatch.md).

## Consequences

### Positive

- **Zero ADR-001 violations.** No build pipeline, single static HTML, GitHub Pages, BYOK, offline-capable — all preserved.
- **Framework-portable.** The `render()` / `update()` / `destroy()` pattern maps to any framework if a future ADR adopts one — mechanical rename, not a rewrite.
- **Minimal failure surface.** Zero dependencies beyond CDN fallbacks for Tailwind, Marked, DOMPurify.

### Negative

- **Manual load order.** Unlike ES modules (which resolve dependency order from `import` statements), classic `<script defer>` tags execute strictly in the order listed in `index.html`. See `docs/technical-notes/file-structure-and-load-order.md`.
- **Manual re-render.** Without a reactivity framework, components must explicitly re-render on state change via `subscribe()`. Manageable at 5–10 components; painful beyond ~50.
- **TypeScript types are documentation-only.** Runtime enforcement relies on the hand-rolled validator in ADR-006, which must be kept in sync with `docs/question-schema.md` by manual discipline.
- **Manual namespace discipline.** Every file wraps its contents in an IIFE and attaches only its intended public surface to `App.*`. Skipping the wrapper leaks internal helpers into the single global scope all classic scripts share.

### Neutral

- **Offline = unstyled-but-functional.** If CDN resources fail, the UI works but looks bare until a v2 critical-CSS block lands. See `docs/technical-notes/cross-cutting-concerns.md`.
- **Model config delegated, not duplicated.** `STABLE_MODELS` / `TEMPERATURE_SUPPORT` live in `docs/model-provider-guide.md` (source of truth) and `scripts/models.js` (mirror); this ADR references them, avoiding staleness.
- **ADR-005 is a future-superseding decision, not a current obligation.** The zero-build approach remains valid until trigger conditions are met.
- **Phase 1→2 storage migration is transparent.** Both phases return `Question[]`; consumers are unaffected. (ADR-004's concern, not this one.)

## References

1. **ADR-001** — App scope: defines the BYOK, no-backend, no-build, no-telemetry, GitHub Pages, and offline-capable constraints this ADR operates within. D1 (this ADR) resolves the "no build pipelines" out-of-scope item.
2. **ADR-002** — AI evaluation reliability: Tier 1 #3 (dead-model recovery → ADR-006 runtime trigger), Tier 1 #4 (`prompt_version` provenance → `docs/technical-notes/prompt-assembly.md`). Implementation details for Tier 2/3 techniques are in `docs/technical-notes/reliability-implementation-notes.md`.
3. **ADR-004** — Question bank and data portability: `file://` support goal that ruled out ES modules; import-time dead-model check (→ ADR-006).
4. **ADR-005** — Build-step escape hatch: the future-superseding decision that would revise D1 if the script count exceeds 15 tags and load-order bugs occur.
5. **ADR-006** — Error recovery and runtime validation: shared `handleApiError()`, `ModelRecoveryBanner` with two trigger points, and the hand-rolled question validator constrained by this ADR's no-framework requirement.

## Technical Notes

The following implementation details are documented in separate reference files:

- **Module loading strategy** — `docs/technical-notes/classic-scripts-not-es-modules.md` (why classic `<script defer>` not `<script type="module">`; the `file://` CORS failure cascade)
- **Component contract** — `docs/technical-notes/component-contract.md` (render/update/destroy lifecycle, factory functions on `App.components`)
- **File structure + load order** — `docs/technical-notes/file-structure-and-load-order.md` (14-file list, `<script defer>` ordering in `index.html`)
- **Baked-in model list** — `docs/technical-notes/model-bakery.md` (mirror of `docs/model-provider-guide.md`)
- **Prompt assembly** — `docs/technical-notes/prompt-assembly.md` (`PROMPT_VERSION`, per-Part prompt injection)
- **Cross-cutting concerns** — `docs/technical-notes/cross-cutting-concerns.md` (state management, rendering safety, CDN fallback)

## Related

- **ADR-001** — App scope and constraints this ADR honors.
- **ADR-002** — AI evaluation reliability, whose Tier 1 #3 drives ADR-006's runtime dead-model trigger.
- **ADR-004** — Question bank and data portability, whose `file://` storage decision drove the classic-script choice.
- **ADR-005** — Build-step escape hatch: would supersede this ADR's vanilla JS / no-build decision.
- **ADR-006** — Error recovery and runtime validation: constrained by this ADR's no-framework, no-build requirement.
