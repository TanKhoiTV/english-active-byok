# ADR-001: App Scope — TOEIC Writing Assistant

## Status

Accepted

## Date

2026-08-15

## Context

The TOEIC Writing Assistant is a browser-based practice tool for the TOEIC Writing section of the English proficiency test. The TOEIC Writing test has **3 parts**:

- **Part 1** (Questions 1–5): Write a single sentence based on a picture, using two given words. (Score 0–3)
- **Part 2** (Questions 6–7): Write a direct email response to a request or inquiry. (Score 0–4)
- **Part 3** (Question 8): Write an opinion essay (~300 words) responding to a statement. (Score 0–5)

The app is **Bring-Your-Own-Key (BYOK)** — users paste their own Google Gemini API key directly into the browser, and no backend server ever handles the key.

### Current State

- **Single static HTML file** (`index.html`) using vanilla JavaScript.
- Loads Tailwind CSS and Marked.js via CDN.
- Uses `localStorage` to persist the API key client-side.
- Provides 1 practice question per Part (3 total).
- Submits responses to the Gemini API and renders feedback as HTML.

### Driving Requirements

1. **Model selection** — Allow users to choose between Gemini models (e.g., 2.5 Flash, 2.5 Pro) rather than hardcoding one.
2. **Question bank per Part** — Group questions by TOEIC Part (1, 2, 3) with a two-level selector (Part → specific question), expanding beyond 1 question per part.
3. **Data portability** — Export user data (API key, question history, model preference) as JSON; import JSON to restore state.
4. **GitHub Pages deployment** — Static hosting only; no backend, no server-side build step.
5. **Offline-capable UI** — The UI (question selection, answer writing, settings) must work even if the CDN resources are unavailable. API evaluation obviously requires connectivity.

### Constraints

- **No backend server.** The app must be a static site (single HTML file or static bundle).
- **Client-side only.** All state (API key, selected model, answer text, evaluation results) lives in the browser.
- **Free hosting.** Must work on GitHub Pages (no custom domain requirements, no serverless functions).
- **BYOK.** API key never leaves the browser; no telemetry, no analytics.

## Decision

**Scope:** The app is a **single-page client-side application** for TOEIC Writing practice. It supports question selection by Part with a hierarchical selector, answer submission with AI evaluation, model selection, and local data persistence with JSON export/import. All logic runs in the browser via CDN-loaded scripts on a static page.

**In scope:**

- Part-based question selector (Part 1, 2, or 3 → specific question from that Part's bank).
- Answer textarea with live word counter.
- Gemini model selection dropdown.
- API key input with save/clear and localStorage persistence.
- AI-powered evaluation with score, error analysis, revision, and recommendations (reliability and rubric approach documented in ADR-002).
- JSON export of user state (API key, model preference, question history).
- JSON import to restore user state.
- Responsive Tailwind-styled UI.
- GitHub Pages deployment as a static HTML file.

**Out of scope:**

- User authentication or accounts (BYOK model).
- Score tracking or leaderboards beyond session history.
- Question answer key verification (feedback is AI-generated only; see ADR-002 for rubric-based reliability measures).
- Mobile-specific optimizations beyond responsive Tailwind classes.
- Server-side rendering, static site generation, or build pipelines.
- Analytics, telemetry, or error reporting.
- Multi-user collaboration or sharing.

## Consequences

### Positive

- **Clear boundaries:** Future feature proposals can be evaluated against this scope to determine if they belong in this app or a separate project.
- **Constraint awareness:** The GitHub Pages + BYOK + no-backend constraints guide future technical decisions.
- **Scalable scope:** The question bank can grow per Part without architectural changes.

### Negative

- **No backend means** no server-side question verification or user progress syncing across devices (beyond JSON export/import).

### Neutral

- The AI evaluation reliability and rubric-based scoring approach is documented in **ADR-002**.
- The architecture decision (which framework to use) is documented in **ADR-003**.
