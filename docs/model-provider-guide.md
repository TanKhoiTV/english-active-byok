# Model Provider Guide

> **Purpose**: Concrete, vendor-specific configuration for AI model access that changes independently of ADR-002's reliability strategy.
>
> **Update rule**: Re-verify before each new app release. Trigger: Tier 1 #3 in ADR-002 ("pin exact model versions + handle dead-model recovery").
> **Last verified**: 2026-08-15
>
> **Scope**: Gemini Developer API only (the app calls `https://generativelanguage.googleapis.com`). This document is structured with `## Gemini` so future providers can be added as additional top-level sections without changing ADR-002.

---

## Gemini

### Current stable model list

| Display name (dropdown) | Pinned model ID | Temperature support | Notes |
| :--- | :--- | :--- | :--- |
| Gemini 3.7 Flash | `gemini-3.7-flash` | No (deprecated) | Latest stable as of 2026-08-15 |
| Gemini 3.6 Flash | `gemini-3.6-flash` | No (deprecated) | Previous stable; may still be available |
| Gemini 3.5 Flash | `gemini-3.5-flash` | No (deprecated) | |
| Gemini 3.5 Flash-Lite | `gemini-3.5-flash-lite` | No (deprecated) | No shutdown date announced |
| Gemini 3.1 Flash-Lite | `gemini-3.1-flash-lite` | No (deprecated) | Scheduled shutdown May 7, 2027; 3.5 Flash-Lite is the recommended replacement |
| Gemini 2.5 Flash | `gemini-2.5-flash` | Yes | GA model; no shutdown date announced |
| Gemini 2.5 Flash-Lite | `gemini-2.5-flash-lite` | Yes | GA model; no shutdown date announced |

[CITATION C3: Google Gemini API docs — Deprecations. "Starting with Gemini 3.6 Flash and Gemini 3.5 Flash-Lite, the sampling parameters temperature, top_p, and top_k are deprecated." https://ai.google.dev/gemini-api/docs/latest-model]

[CITATION C3: Gemini API changelog, July 21, 2026. "The sampling parameters temperature, top_p, and top_k have been deprecated across the Gemini model suite."]

**⚠️ Verify at implementation time**: This list was accurate as of 2026-08-15. Google releases new stable models frequently (3.7 Flash went stable within days of this writing). Call `GET https://generativelanguage.googleapis.com/v1beta/models` with the app's API key to discover the current list, and select only entries that are (a) stable and (b) listed above. Remove any model IDs from the dropdown that no longer appear in the list.

### Temperature support

- **Gemini 3.x models** (3.7, 3.6, 3.5, 3.1 Flash-Lite): Temperature, top_p, and top_k are **deprecated**. These parameters are **ignored** by the API. Including them does not cause an error on current models, but Google states **future model generations will return HTTP 400**. [CITATION C3: https://ai.google.dev/gemini-api/docs/latest-model]
- **Gemini 2.5 Flash / Flash-Lite (GA)**: Temperature is **supported**. [CITATION-NOTE: Verify this at implementation time — the deprecation may extend to 2.5 GA models in future updates.]
- **Gemini 2.0 series**: **Shut down** June 1, 2026. [CITATION C3: https://ai.google.dev/gemini-api/docs/deprecations]

**Determinism strategy for Gemini**:

- On Gemini 3.x: Rely on **system instructions** + **responseSchema** only. Do not set temperature in `generationConfig` (it will be ignored).
- On Gemini 2.5 (if still available): Set `temperature: 0.2` in `generationConfig` as a secondary safety net.

[CITATION C3: Google Gemini API docs — "To ensure determinism, developers should use system instructions with explicit rules instead of adjusting these sampling parameters." https://ai.google.dev/gemini-api/docs/latest-model]

### Structured output mechanism

Use `responseMimeType: "application/json"` with a `responseSchema` in `generationConfig`. [CITATION C1: Google Gemini API docs — Structured Output. https://ai.google.dev/gemini-api/docs/generate-content/structured-output]

```json
{
  "generationConfig": {
    "responseMimeType": "application/json",
    "responseSchema": {
      "type": "object",
      "properties": {
        "criteria": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "name": { "type": "string" },
              "score": { "type": "integer" },
              "notes": { "type": "string" }
            },
            "required": ["name", "score", "notes"]
          }
        },
        "final_score": { "type": "integer" },
        "band_justification": { "type": "string" },
        "error_analysis": { "type": "string" },
        "polished_revision": { "type": "string" },
        "key_recommendations": {
          "type": "array",
          "items": { "type": "string" }
        },
        "model_used": { "type": "string" },
        "prompt_version": { "type": "string" },
        "timestamp": { "type": "string" }
      },
      "required": ["criteria", "final_score", "band_justification"]
    }
  }
}
```

### Model naming and aliases

- **Stable**: Bare name, no suffix (e.g., `gemini-3.7-flash`) — does not change. Use these for the dropdown.
- **Preview**: Dated string (e.g., `gemini-3.5-flash-preview-09-2025`) — may be deprecated with 2+ weeks' notice.
- **Latest alias**: Version-agnostic `gemini-{model}-latest` pattern (e.g., `gemini-flash-latest`) — updated automatically. **Not for production use.**

[CITATION C2: Google Gemini API docs — Model version name patterns. https://ai.google.dev/gemini-api/docs/models]

**Note**: The `-002` numeric suffix convention (e.g., `gemini-1.5-flash-002`) was used in the Gemini 1.5 era and appears in some cookbooks [CITATION C8: https://github.com/doggy8088/gemini-api-cookbook], but does **not** apply to the current Gemini Developer API model naming for 2.x/3.x.

### Deprecation and sunset schedule

| Model family | Status | Sunset / Shutdown date |
| :--- | :--- | :--- |
| Gemini 2.0 Flash / Flash-Lite | Shut down | June 1, 2026 |
| Gemini 2.5 Flash / Flash-Lite (GA) | Available | No shutdown date announced |
| Gemini 2.5 preview / image sub-variants | Various | November 2025 – October 2026 (staggered) |
| Gemini 3.1 Flash-Lite | Scheduled | May 7, 2027 (gemini-3.5-flash-lite is the recommended replacement) |
| Gemini 3.5 Flash-Lite | Available | No shutdown date announced |

[CITATION C3: Google Gemini API docs — Deprecations. https://ai.google.dev/gemini-api/docs/deprecations]

**⚠️ Verify at implementation time**: This schedule changes as Google updates deprecation announcements. Check `https://ai.google.dev/gemini-api/docs/deprecations` before each release.

### Dead-model recovery

If a user's imported JSON references a model ID that has been shut down:

1. The `evaluateAnswer()` fetch call will receive an HTTP 4xx error from `generativelanguage.googleapis.com`.
2. Catch the error and display: "The model in your exported settings is no longer available. Please select a different model above."
3. Pre-select the first stable model from the current list (see § Current stable model list) in the dropdown for one-click recovery.

## References

- C1: <https://ai.google.dev/gemini-api/docs/generate-content/structured-output>
- C2: <https://ai.google.dev/gemini-api/docs/models>
- C3: <https://ai.google.dev/gemini-api/docs/latest-model> ; <https://ai.google.dev/gemini-api/docs/deprecations> ; <https://ai.google.dev/gemini-api/docs/changelog>
- C8: <https://github.com/doggy8088/gemini-api-cookbook>
