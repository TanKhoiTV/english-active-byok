# Changelog

All notable changes are documented here. Format follows
[Keep a Changelog](https://keepachangelog.com/); the *app* uses semantic versioning
for releases. The evaluation prompt carries its own independent `PROMPT_VERSION`
(see `index.html`), recorded in
`docs/technical-notes/prompt-assembly.md` § Prompt version history.

## [1.0.0] - 2026-08-16

First tagged release. Single-file, no-build, BYOK static app deployed to GitHub
Pages.

### Added

- Original TOEIC Writing question bank: **30 questions (10 per Part 1/2/3)**,
  embedded as inline JSON (`#questionBank`), `file://`-safe (no `fetch`).
- AI evaluation via the Gemini API (BYOK) with structured JSON output
  (`responseSchema`), per-Part rubric criteria, and deterministic-scoring rules.
- Per-criterion feedback (score + notes), band justification, error analysis,
  polished revision, and key recommendations.
- Provenance line (`model_used` / `prompt_version` / `timestamp`) plus an
  **"AI estimate — not an official TOEIC score"** disclaimer.
- Persistent cross-reload history (localStorage, capped at 50).
- Settings + history export/import with import-time dead-model check.
- Four-class API error handling (`handleApiError`): `429` / `400·404` /
  `401·403` / network, surfacing the raw message.
- Init-time validator that throws on a malformed question bank.
- Model selector (5 stable Gemini models; default `gemini-3.6-flash`), with
  `temperature` intentionally omitted (deprecated on Gemini 3.x).
- iPadOS Safari a11y polish (text selection, focus release on outside tap) and
  `<label>` associations.

### Changed

- **Part 2 directive & rubric standardization (ADR-004 / ETS alignment):**
  consistent "two pieces of information + one question" directives; rubric
  aligned to ETS 0–4 holistic scoring (commits `9e80a93`→`0e54f61`).
- **`PROMPT_VERSION` advanced to `1.1.0`** to reflect the Part 2 refactor, which
  had previously shipped under `1.0.0` due to a versioning oversight.

### Governance / decisions (this cycle)

- **P3 structural split — decided to keep the monolithic `index.html`.** The
  ADR-004 size trigger (~30 questions / 50 KB) is met, but the ADR-003/ADR-005
  structural trigger (> 15 `<script>` tags AND load-order bugs) is not
  approached. Re-evaluate only on the load-order trigger. (ROADMAP.md § P3)
- **Human-review gate cleared** for the 30 assistant-authored prompts
  (pre-release pass 2026-08-16).
- **Model re-verification** against `docs/model-provider-guide.md` passed:
  `MODELS` in sync (5 entries, default `gemini-3.6-flash`); `temperature`
  omitted from `generationConfig`.

### Known limitations / deferred

- `ModelRecoveryBanner` (ADR-006) remains deferred; dead-model recovery is
  handled at import time + classified error messages only.
- ADR-003 14-file split / ADR-005 build step deferred (keep monolith).
- No automated test harness yet.
