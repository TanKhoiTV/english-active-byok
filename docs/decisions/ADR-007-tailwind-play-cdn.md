# ADR-007: Tailwind Play CDN for styling (no-build; accept production warning, no SRI)

## Status

Accepted

## Date

2026-08-16

## Context

The app is a zero-build, single-file static page (ADR-003 D1 classic scripts; ADR-005
zero-build architecture; ADR-004 Phase 1 embedded data) that must run from both `file://`
and HTTPS. Styling uses Tailwind utility classes delivered via the Tailwind Play CDN
(`https://cdn.tailwindcss.com`).

The Play CDN is a development/prototyping tool. Using it in this app produces two known
consequences:

1. **Production warning** — the CDN script prints
   `cdn.tailwindcss.com should not be used in production…` to the console on every load.
   It is informational only; styling still works. Root cause: the CDN ships the full
   Tailwind engine and compiles CSS in the browser at runtime (large payload, possible
   flash of unstyled content), rather than serving a pre-built stylesheet.
2. **No Subresource Integrity (SRI) possible** — SRI requires the resource to be
   CORS-served (`crossorigin="anonymous"` + `integrity`). The Play CDN returns **no**
   `Access-Control-Allow-Origin` header (verified via `curl -I`), so adding
   `crossorigin`+`integrity` makes the browser block the script under any origin,
   including `file://` (`net::ERR_FAILED`). Therefore Tailwind cannot carry an SRI hash.
   (marked and DOMPurify are loaded from jsDelivr, which *does* send
   `access-control-allow-origin: *`, so they keep `integrity`+`crossorigin`.)

The no-build constraint (ADR-005) rules out a Tailwind build step, which is the usual way
to avoid both issues.

## Decision

Keep the Tailwind Play CDN, **pinned to a fixed version (`3.4.17`)**, as the styling
delivery mechanism.

- **Accept** the Play CDN's "not for production" console warning as an informational,
  harmless message — it does not affect functionality and is the expected cost of a
  zero-build single-file design.
- **Do not** add `integrity`/`crossorigin` to the Tailwind `<script>` (it would break
  loading). Instead, pin the version for supply-chain determinism: a bumped or compromised
  CDN URL is at least visible in the committed file and in `git diff`.
- marked and DOMPurify remain pinned + SRI via jsDelivr.
- The CORS/SRI limitation is also noted inline in `index.html` (head comment above the
  Tailwind tag) so the constraint is visible at the code site.

## Alternatives Considered

### Precompile Tailwind to static CSS (CLI) and inline/link locally

- Pros: removes the production warning, enables SRI (same-origin), faster first paint (no
  in-browser compile), smaller payload.
- Cons: reintroduces a build step (Tailwind CLI) — directly contradicts ADR-005's
  zero-build architecture; emits a build artifact (compiled CSS) that must be regenerated
  whenever classes change; either breaks the single-file shape (vendored `tailwind.css`)
  or requires inlining a large `<style>` block.
- Rejected (for now): would require a superseding ADR to ADR-005. Revisit if the app is
  ever deployed at scale or the warning becomes unacceptable.

### Switch to a CORS-capable Tailwind delivery (e.g. `@tailwindcss/browser` v4)

- Pros: could permit SRI.
- Cons: v4 differs from the v3 utility/config model the app uses; larger integration
  change; still a runtime-compiled browser bundle (keeps the production-warning class of
  issue).
- Rejected: not worth the churn for a local practice tool.

### Drop the version pin (use unpinned `cdn.tailwindcss.com`)

- Cons: loses supply-chain determinism; a future Tailwind release could change behavior
  silently.
- Rejected: pinning (`3.4.17`) is nearly free and preserves visibility of CDN changes.

## Consequences

### Positive

- Zero-build, `file://`-compatible, single-file design preserved (consistent with
  ADR-003 / ADR-004 / ADR-005).
- No toolchain, lockfile, or CI dependency.

### Negative

- Informational "not for production" console warning on every load (harmless).
- Tailwind is the one dependency that cannot carry SRI; mitigated only by version pinning,
  not by a hash check.

### Neutral

- marked / DOMPurify keep full SRI via jsDelivr.
- Inline head comment in `index.html` records the CORS/SRI limitation at the code site.

## References

1. **ADR-003** — Framework and component architecture (classic `<script defer>`, no-build):
   the decision this styling choice depends on.
2. **ADR-005** — Build-step migration escape hatch: the no-build constraint that rules out
   precompiling Tailwind.
3. **ADR-004** — Question bank and data portability: Phase 1 single-file deployment
   requirement.
4. `docs/technical-notes/classic-scripts-not-es-modules.md` — the `file://` CORS failure
   cascade that also explains why `crossorigin` on the Tailwind tag breaks loading.

## Related

- ADR-003, ADR-004, ADR-005
