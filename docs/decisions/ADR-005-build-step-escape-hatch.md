# ADR-005: Build-Step Escape Hatch

## Status

Proposed

## Date

2026-08-16

## Context

ADR-003 D1 chose classic `<script defer>` tags (not ES modules — see `docs/technical-notes/classic-scripts-not-es-modules.md`) to support `file://` access alongside HTTPS deployment (ADR-004 Phase 1/2). This zero-build approach is a deliberate tradeoff for shipping velocity at small scale. The **current** app has ~9 scripts and loads reliably in both deployment modes.

The planned restructure (ADR-003) targets 14 script files — documented in `docs/technical-notes/file-structure-and-load-order.md`. That count is tracked against this ADR's 15-tag trigger threshold; the restructure itself stays under the cliff, but the margin is intentional so that even one additional file after restructure will trigger a revisit.

However, a zero-build architecture has a scalability cliff. At 15+ interdependent files, manual load-order management in `index.html` becomes error-prone — there is no automatic dependency resolution to catch when a new file depends on one declared later in the `<head>`.

## Decision

The zero-build architecture remains valid **until** a future ADR explicitly decides to adopt a build step. When that future ADR is written, it must amend both:

- **ADR-001** — remove "no server-side build step" from the driving requirements and remove "build pipelines" from out-of-scope.
- **ADR-003 D1** — revise the vanilla JS / classic-scripts decision to permit ES module authoring (see `docs/technical-notes/classic-scripts-not-es-modules.md`).

**Trigger conditions** (both must be met):

1. The `<head>` script list exceeds 15 `<script defer>` tags, AND
2. A load-order bug is encountered in two consecutive features.

Rationale: at ~15+ interdependent files, manual ordering becomes error-prone and a bundler's dependency graph becomes worth the toolchain cost.

## Migration Path

When triggered, the future superseding ADR may adopt ES module authoring bundled to a **single classic `<script>` output** (not `type="module"`). This preserves `file://` compatibility — the core concern that ruled out ES modules in ADR-003 D1 — while gaining `import`/`export` syntax and automatic dependency resolution.

### Implementation steps (for the superseding ADR to specify)

1. Add `esbuild` as a dev dependency (or Vite with `--bundle`).
2. Create `src/` with ES module source files using native `import`/`export`.
3. Configure the bundler to output a single classic `<script>` IIFE targeting the existing `window.App` namespace.
4. Replace the 15 `<script defer>` tags in `index.html` with the single bundled output.
5. Retain the inline `window.App = { components: {} }` bootstrap (runs at parse time before the bundle).

## Consequences

### Positive

- **Explicit guardrail:** The zero-build approach won't be abandoned prematurely — two concrete conditions must be met.
- **Preserves `file://` support:** The bundler targets a single classic script, sidestepping the CORS failure mode that motivated ADR-003 D1 (documented in `docs/technical-notes/classic-scripts-not-es-modules.md`).
- **No current cost:** This ADR is a placeholder; no code changes until the trigger conditions are met.

### Negative

- **Adds a future decision point:** When triggered, the team must write a superseding ADR with new trade-offs (lockfile, CI dependency, toolchain complexity).

### Neutral

- **Current app is safe:** The app currently has ~9 scripts and has not hit the 15-tag or load-order-bug threshold.
- **Bundler choice is deferred:** Whether to use esbuild, Vite, or another tool is a decision for the superseding ADR, not this one.

## References

1. **ADR-001** — App Scope: defines the "no server-side build step" constraint that this escape hatch would require amending.
2. **ADR-003 D1** — Vanilla JS / classic `<script defer>`, not ES modules: the decision this escape hatch would supersede. See `docs/technical-notes/classic-scripts-not-es-modules.md` for the full `file://` CORS failure cascade.
3. **ADR-003** (file structure — see `docs/technical-notes/file-structure-and-load-order.md`) — the script-count metric is tracked in that reference doc.
4. **ADR-004** — Question bank and data portability: Phase 1/2 storage requirements that the classic-script approach satisfies.

## Related

- **ADR-001** — App scope and constraints
- **ADR-003** — Framework and component architecture (primary; D1 is revised by this future ADR)
- **ADR-004** — Question bank and data portability
- **ADR-007** — Tailwind Play CDN: the canonical no-build styling tradeoff (production warning, no SRI) this escape hatch accepts
