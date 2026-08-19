# Roadmap

Single source of truth for **current state** and **next objectives** — deferred or
not-yet-implemented work. Derived from the ADRs, which remain authoritative on *how*
things should work. Governance: `index.html` is the current implementation of
record; `docs/` is the authoritative spec (see `CONTRIBUTING.md`). When the two
conflict, docs win as design and `index.html` is brought into conformance.

## Current state

- `index.html` — monolithic single-page app; no build step (ADR-001, ADR-003 D1).
- Question bank — 30 embedded questions (10 per Part), ADR-004 Phase 1 (inline JSON in `index.html`); **at** the ~30-question / 50 KB split trigger — `index.html` is now ~82 KB, so the ADR-003/ADR-005 trigger (P3) is met.
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
| ADR-001 / ADR-004: export/import settings + `answerHistory` | **Implemented** (P2 #2/#3) — client-side export/import via Blob + `FileReader`; `answerHistory` persists under `gemini_eval_history`. Unblocks ADR-006 *import-time* dead-model check |

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

**Work landed after the last roadmap sync (`6d26d4e`)** — for the record; the
P2 items below are already marked DONE, and the Part 2 refactor is newly captured:

- Question-bank batch 2 (`93b7dc2`): grew the embedded bank 18 → 30 questions
  (10 per Part); all pass `validateQuestion`. Pushes the ADR-003/ADR-005 split
  trigger (see P3).
- iPadOS Safari a11y polish (`dd10536`, `d27d8ec`, `281c3f3`): enabled text
  selection on the question statement + analysis regions, and released the
  `<textarea>` focus on outside tap so the keyboard dismisses.
- **Part 2 directive & rubric standardization (ADR-004 / ETS alignment)**
  (`9e80a93`, `0e19371`, `ae4d925`, `5328e03`, `0e54f61`): reworked every Part 2
  prompt's `directives` into a consistent *"give TWO pieces of information + ask
  ONE question"* shape, collapsed redundant D2/D3 directives, softened overly
  explicit asks, aligned the Part 2 rubric/scoring anchors with ETS 0–4 holistic
  scoring, and corrected directive grammar/capitalization. Supersedes the ad-hoc
  p2-004 copy-edit (`6d26d4e`) noted in P2 #1 and resolves that specific review
  item; the standing human-review gate for unreleased assistant-authored prompts
  still applies before any release.

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
2. **Release policy (Tier 1 #3) — DONE (process established).** The `MODELS` list in
   `index.html` was re-verified against `docs/model-provider-guide.md` on 2026-08-16
   and is **in sync** (same 5 entries; default `gemini-3.6-flash`). `temperature` is
   deprecated on Gemini 3.x and is **intentionally omitted** from every
   `generateContent` request — confirmed in `evaluateAnswer`'s `generationConfig`
   (no `temperature` field; the `temp` flag on `MODELS` is display-only and never
   sent). A `LAST_VERIFIED` note above `MODELS` plus the checklist below make this a
   repeatable pre-release step rather than a one-off.

### Pre-release model re-verification checklist (P1 #2 / ADR-002 Tier 1 #3)

Run before every release:

1. Open `docs/model-provider-guide.md` and confirm the **Current stable model list**
   and **Deprecation and sunset schedule** are current (re-check
   <https://ai.google.dev/gemini-api/docs/deprecations> if anything looks stale).
2. Diff that list against the `MODELS` array in `index.html` — every stable entry must
   appear, and no removed/deprecated model should remain in the dropdown.
3. Confirm `DEFAULT_MODEL` still points at the intended default (currently
   `gemini-3.6-flash`).
4. Confirm `temperature` is **not** present in `evaluateAnswer`'s `generationConfig`
   (Gemini 3.x deprecates it; keep omitted).
5. Update the `LAST_VERIFIED` comment above `MODELS` with today's date.
6. *(Optional, needs a Gemini API key)* Hit `GET …/v1beta/models` to confirm the live
   catalog still matches; drop any model ID no longer returned.

### P2 — Feature growth & UX

1. **Question-bank growth (ADR-004)** — **DONE (batches 1–2).** Grew the embedded,
   original question bank from 9 → 18 → 30 questions (10 per Part; all pass
   `validateQuestion`). Stays embedded as JSON in `#questionBank` (parsed
   synchronously, file://-safe — no `fetch`), consistent with ADR-004 Phase 1 and
   ADR-001. The `questions.json` + `fetch`/cache split stays deferred with
   ADR-003/ADR-005, but the ~30-question / 50 KB trigger is now **reached** (see
   P3). New prompts are assistant-authored and still need the human review gate
   before a release. The 2026-08-16 p2-004 role-mismatch copy-edit (`6d26d4e`) was
   folded into the Part 2 directive standardization refactor captured below, which
   resolves that specific review item; the standing human-review gate for
   unreleased assistant-authored prompts still applies before any release.
2. **Persistent cross-reload history** (localStorage) — **DONE.** Each successful
   evaluation is saved (prompt, answer, structured feedback, model, calibration flag,
   prompt version) under `gemini_eval_history` and survives a reload; a collapsible
   "History & Backup" panel lists records newest-first and expands to show the full
   evaluation (reusing `renderEvaluation`). Capped at `HISTORY_LIMIT = 50`.
3. **Export/import settings + history (ADR-001/ADR-004)** — **DONE.** "Download
   backup" exports `{ backupVersion, exportedAt, promptVersion, settings, history }`
   as a JSON Blob (API key included only on opt-in). "Restore backup" reads it via
   `FileReader`, merges history by `id`, restores the model, and runs ADR-006's
   *import-time* dead-model check: any model not in `MODELS` is flagged and the user
   is offered a switch to `DEFAULT_MODEL` (`gemini-3.6-flash`); affected history rows
   are marked "⚠ deprecated model". No `fetch`; everything stays client-side.

### P3 — Structural (trigger now reached)

1. **ADR-003 14-file split / ADR-005 build step.** Trigger **reached**: the bank is
    now 30 questions and `index.html` is ~82 KB, meeting the ~30-question / 50 KB
    split trigger noted in P2 #1. **Still deferred by decision** — the 14-file split
    sits *below* ADR-005's 15-script-tag cliff and would force an ADR-005 supersede,
    and no load-order pain or script-tag-count excess has appeared yet. Re-evaluate
    the split only when load-order pain actually appears or the script-tag count is
    exceeded; otherwise keep the monolith.

## Watch list

- `docs/reference-library.md` → "Future Sources" — research items to track.

## Suggested sequence

P0 (1–4) as one cohesive `index.html` change → P1 (5–6) as needed → P2 (7–9) by
product priority → P3 only on trigger. Do **not** start the ADR-003/005 split
independently.
