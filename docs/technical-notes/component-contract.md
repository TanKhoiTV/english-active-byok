# Component Contract

> Technical reference — component contract decided by ADR-003.

## Decision

Each component is a factory function attached to `App.components`, exposing a `render()` / `update(props)` / `destroy()` lifecycle. State flows one direction: `AppState` → props → `render()`.

## Alternatives Considered

### Framework component classes (React, Vue, Lit)

Rejected: pulls in a framework dependency and a build/compile step, violating ADR-003 D1. Not justified for 5–10 components with infrequent UI updates.

### Vanilla DOM methods (`querySelector` + imperative updates)

Rejected: no standard lifecycle, state scattered across DOM nodes, difficult to extend systematically.

### Custom elements (Web Components)

Rejected: heavier than needed; the `render`/`update`/`destroy` contract serves the same portability goal with less ceremony.

## Rationale

This is a minimal emulation of a framework component contract — enough to decompose features, and portable to Alpine/Lit/React later. The single source of truth for component state is `App.store.AppState`, not scattered DOM or local variables.

## Components Defined

The five components referenced by ADR-001 (in scope) and ADR-003 (file structure):

- `App.components.ModelDropdown` — renders from `App.models`
- `App.components.QuestionSelector` — Part → question hierarchical selector
- `App.components.AnswerEditor` — answer textarea with live word counter
- `App.components.ModelRecoveryBanner` — `showModelRecovery(deprecatedModelId, onReselect)`
- `App.components.ExaminerOutput` — renders evaluation output (marked + DOMPurify)

## Cross-reference

- **ADR-001:** Defines the component feature requirements (selector, editor, dropdown, output)
- **ADR-003:** State management, rendering safety, CDN fallback (see `cross-cutting-concerns.md`)
