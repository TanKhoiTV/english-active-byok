# Baked-in Model List

> Technical reference — model list mirror for ADR-003 and ADR-006.

## Decision

The model dropdown renders from `App.models.STABLE_MODELS` and `App.models.TEMPERATURE_SUPPORT`, which are a runtime mirror of `docs/model-provider-guide.md`. The guide is the source of truth (human-readable, re-verified before each release per ADR-002 Tier 1 #3); `scripts/models.js` is the copy the UI reads.

## Alternatives Considered

### Live `models.list()` at runtime

Rejected: would fail offline and empty the dropdown, violating ADR-001's offline-capability constraint. A v2 "refresh models" button could augment the baked-in list, but never replace it.

## Rationale

Offline-by-default is non-negotiable per ADR-001. The model list shared between the guide and `scripts/models.js` must be re-synced before each release — this is a known, intentional cost, already a checklist item per ADR-002 Tier 1 #3. See `docs/model-provider-guide.md` for the current stable lineup, temperature support per model family, and deprecation/sunset schedule.

## Cross-reference

- **ADR-001:** Offline-capable UI constraint
- **ADR-002 Tier 1 #3:** Pin exact model versions + dead-model recovery
- **ADR-006:** `handleApiError()` uses `STABLE_MODELS` for dead-model detection

## Note on dual mirrors

This file (`model-bakery.md`) is the **documentation** mirror — a written companion describing the runtime mirror. The runtime mirror is `scripts/models.js` (listed in `docs/technical-notes/file-structure-and-load-order.md`), which is the code the dropdown actually reads. Both mirror `docs/model-provider-guide.md` (source of truth). Sync is a **manual pre-release checklist item** (ADR-002 Tier 1 #3): before each release, the curator checks the guide and updates both mirrors — no automated sync mechanism exists.
