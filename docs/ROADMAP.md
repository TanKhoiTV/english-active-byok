# Roadmap

Single source of truth for **current state** and **next objectives** — deferred or
not-yet-implemented work. Derived from the ADRs, which remain authoritative on *how*
things should work. Governance: `index.html` is the current implementation of
record; `docs/` is the authoritative spec (see `CONTRIBUTING.md`). When the two
conflict, docs win as design and `index.html` is brought into conformance.

## Current state

- `index.html` — monolithic single-page app; no build step (ADR-001, ADR-003 D1).
- Question bank — 9 embedded questions, ADR-004 Phase 1 (inline JSON in `index.html`).
- Model selection dropdown — **done**; inline `MODELS` mirror of
  `docs/model-provider-guide.md`; defaults to **Gemini 3.6 Flash** (Gemini 2.5
  Flash / 2.5 Flash-Lite removed — both return HTTP 404 as of 2026-08-16);
  persisted in `localStorage`.

### Conformance status (accepted ADRs vs. `index.html`)

| Accepted requirement | Status in `index.html` |
| --- | --- |
| ADR-006: `handleApiError()` four-class classification (429 / 400·404 / 401·403 / network) + raw API message surfaced | **Implemented** (P0, commit `055c508`; raw-message rephrased `8c01de8`) |
| ADR-006: `ModelRecoveryBanner` dead-model recovery (HTTP 400/404 → reselect) | **Removed / deferred** — banner element deleted in `8c01de8` to fix header layout; error classification + raw message retained. Re-add after layout settled. |
| ADR-006: init-time validator **throws** on authoring mistakes | **Implemented** (P0, commit `055c508`) |
| ADR-002 T1#1: structured output (`responseSchema`) + per-criterion decomposition | **Implemented** (P0, commit `055c508`) |
| ADR-002 T1#4: `PROMPT_VERSION` + provenance + "AI estimate" disclaimer | **Implemented** (P0, commit `055c508`) |
| ADR-002 T2: few-shot calibration anchors | **Implemented** (P1) — opt-in, loaded from gitignored `docs/questions-sample.json` via a local file picker (never fetched/embedded) |
| ADR-001 / ADR-004: export/import settings + `answerHistory` | Not started — blocks ADR-006 *import-time* trigger (P2) |

**Working before P0:** per-Part content-coverage signals (`targetWords`,
`directives`, `title`/`essayTopic`) via `buildQuestionText()`; deterministic
system-instruction rules; model version pinning + dropdown.

## Next objectives

### P0 — Close the ADR conformance gap — DONE (commit `055c508`)

All contained to `index.html`; no new dependencies, no build step.

1. **ADR-002 T1#1 — structured output + criterion decomposition.**
   - Add `generationConfig: { responseMimeType: "application/json", responseSchema: {...} }`
     to the `evaluateAnswer()` `fetch` body, using the canonical schema in
     `docs/model-provider-guide.md` (§ Structured output mechanism):
     `criteria[]` (name/score/notes), `final_score`, `band_justification`,
     `error_analysis`, `polished_revision`, `key_recommendations[]`, plus
     `model_used` / `prompt_version` / `timestamp`.
   - Rewrite `SYSTEM_PROMPT` to emit that JSON (keep the existing per-Part rubric
     criteria + deterministic-scoring rules).
   - Render each field; run `marked` + `DOMPurify` on the free-text fields
     (`error_analysis`, `polished_revision`, `key_recommendations`) as today.
   - Per-Part sub-criteria to enforce in `responseSchema` (from
     `docs/technical-notes/reliability-implementation-notes.md`):
     Part 1 → grammar, task_completion, word_usage (0–3); Part 2 →
     sentence_quality_variety, vocabulary, organization (0–4); Part 3 →
     thesis_and_support, grammar, vocabulary, organization (0–5).

2. **ADR-002 T1#4 — provenance + disclaimer.**
   - Add a `PROMPT_VERSION` constant (bump on any prompt change).
   - Capture `model_used` / `prompt_version` / `timestamp` from the response and
     render a subtle provenance line.
   - Render a visible **"AI estimate — not an official TOEIC score"** banner above
     the evaluation output.

3. **ADR-006 — error recovery.**
   - Extract `handleApiError(err, res)` classifying the four error classes into one
     actionable message each: `429` (rate limit → retry shortly), `400/404`
     (dead/invalid model → recovery), `401/403` (key invalid → re-enter key),
     `network` (connectivity → generic). Raw API errors never reach the user.
   - Add a `ModelRecoveryBanner` inline element: on `400/404`, pre-select the newest
     stable model in `#modelSelect` for one-click recovery, with the copy
     *"The model in your exported settings is no longer available. Please select a
     different model above."* (matches `docs/model-provider-guide.md` § Dead-model
     recovery).
   - Wire `handleApiError` into the `evaluateAnswer()` `catch` (replace the raw
     `err.message` render).

4. **ADR-006 — init-time dev guard.**
   - Make `validateQuestion()` **throw** on failure during `loadQuestions()`
     (catches authoring bugs before first render). Keep graceful degradation for
     the future import-time user-data path.

**Verification (no test harness exists):** open `index.html` via `file://`, run one
question from each Part with a valid key; confirm structured output renders, the
disclaimer + provenance show, and that an invalid model id / bad key produces the
classified banner rather than a raw error.

**Follow-up fixes completed after `055c508` (still P0 scope, no open items):**

- Dead-model fix (`cb3b926`): removed `gemini-2.5-flash` /
  `gemini-2.5-flash-lite` from `MODELS` (both return HTTP 404 as of 2026-08-16)
  and defaulted to `gemini-3.6-flash`.
- Header UI cleanup (`8c01de8`, `10a32f9`, `d944d36`, `8a7ac7c`): dropped the top
  `#recoveryBanner`, kept `#keyStatus` as a right-aligned block below the controls,
  render `Estimated Score: X/max` and per-criterion `score/max`, and rephrased the
  bottom error text to classify (key / model / rate-limit / network) and show the
  raw API message.
- a11y (`9dc7e92`): associated `Select Question Prompt` → `#questionSelect` and
  `Your Written Response` → `#userInput`.

### P1 — Reliability hardening (optional, lower priority)

1. **ADR-002 T2 — few-shot calibration anchors** — **DONE.** Opt-in. ~3 examples
   per Part (high/mid/low) are authored as ORIGINAL paraphrased scoring samples
   inside the gitignored `docs/questions-sample.json` (`calibrationAnchors` section,
   copyright-safe per ADR-004 Quality Gate). At runtime the user loads that file
   locally via a file picker (`FileReader`) — `index.html` never references or
   `fetch()` the file, and the content is never embedded, uploaded, or persisted
   (ADR-004 hard rule). When enabled, `buildCalibrationSection(part)` appends the
   Part's anchors to `SYSTEM_PROMPT` for that evaluation only. **Refresh the anchors
   per release** to match the current prompt version. The on-disk file is still
   gitignored, so it is never committed to the public Pages repo.
2. **Release policy (Tier 1 #3):** re-verify the `MODELS` list in `index.html`
   against `docs/model-provider-guide.md` before each release; the app intentionally
   omits `temperature` (deprecated on Gemini 3.x) — keep it that way.

### P2 — Feature growth & UX

1. **Question-bank growth (ADR-004).** Add original questions (copy-edit + human
   review gate) toward the ~30-question / 50 KB trigger for Phase 2
   (`questions.json` + `fetch`/cache). Export/import stays additive.
2. **Persistent cross-reload history** (localStorage). Currently deferred but high
   user value; natural once structured output exists (store prior evaluations with
   provenance).
3. **Export/import settings + history (ADR-001/ADR-004).** Also unblocks ADR-006's
   *import-time* dead-model check (`App.exportImport.deserialize()` referencing
   `STABLE_MODELS`).

### P3 — Structural (only on trigger)

1. **ADR-003 14-file split / ADR-005 build step.** Stay deferred. The 14-file
    split sits *below* ADR-005's 15-script-tag cliff and would force an ADR-005
    supersede. Schedule only when load-order pain actually appears or the script-tag
    count is exceeded.

## Watch list

- `docs/reference-library.md` → "Future Sources" — research items to track.

## Suggested sequence

P0 (1–4) as one cohesive `index.html` change → P1 (5–6) as needed → P2 (7–9) by
product priority → P3 only on trigger. Do **not** start the ADR-003/005 split
independently.
