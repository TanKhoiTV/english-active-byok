# TOEIC Writing Format Reference

Canonical, app-internal description of the TOEIC Writing test format, per Part. This document aligns three things: the question schema ([`question-schema.md`](question-schema.md)), how questions are authored, and what the evaluator prompt asks the AI examiner to check.

This is an **original summary of the public task structure**. It does **not** reproduce any copyrighted ETS questions. The only place ETS-originals live is the internal calibration file [`questions-sample.json`](questions-sample.json), which is gitignored and never deployed (see ADR-004).

## Overview

ETS does not publish a topic syllabus; the "scope" is defined by the three task *formats*, not by a list of subjects. Topics fall within "daily life and the global workplace." The evaluation criteria per Part are fixed and are mapped field-by-field in [`question-schema.md`](question-schema.md).

## Part 1 — Write a sentence based on a picture (Score 0–3)

- **Task:** write ONE sentence describing a picture, using TWO given words or phrases.
- **What the response must contain:** both target words/phrases, integrated naturally; a sentence that addresses the depicted scenario.
- **Evaluation:** `grammar`, `task_completion` (relevance + both words used), `word_usage`.
- **Schema fields:** `targetWords` (exactly 2), optional `imageAlt` describing the depicted scene.

## Part 2 — Respond to a written request (Score 0–4)

- **Task:** read a workplace email and write a response.
- **What the response must contain:** address the email's specific requests, **and typically include two pieces of information (e.g., two reasons or two details) and ask one question of the sender**. The exact required elements are captured per-question in the `directives` array.
- **Why "two + one":** the ETS Part 2 scoring rewards responses that fully answer the prompt's directions and demonstrate sentence variety and organization. The "two reasons + one question" shape is the standard structure used by the official sample tasks and by prep materials; it is what a realistic TOEIC Part 2 response is expected to contain.
- **Evaluation:** `sentence_quality_variety`, `vocabulary`, `organization`.
- **Schema fields:** `emailContext` (`from` / `to` / `subject` / `sent` / `body`) plus an optional `directives` array describing what the response should include. The UI renders `directives` as a "Your response should include:" list, and the evaluator receives them so feedback can check coverage.

## Part 3 — Opinion essay (Score 0–5)

- **Task:** state, explain, and support an opinion on a workplace issue; roughly 300 words.
- **What the response must contain:** a clear thesis (opinion), supported with reasons and examples, in organized paragraphs.
- **Evaluation:** `thesis_and_support`, `grammar`, `vocabulary`, `organization`.
- **Schema fields:** `essayTopic` (the prompt) plus an optional `title` (a short display title, like an email subject) and optional `wordTarget`.

## Scoring bands (fixed by ETS — do not change)

| Part | Task | Score Range |
| --- | --- | --- |
| Part 1 | Write a sentence based on a picture | 0–3 |
| Part 2 | Respond to a written request | 0–4 |
| Part 3 | Write an opinion essay | 0–5 |

## Authoring notes

- Follow the *format*, not the *wording*, of any ETS sample. Questions must be originals (ADR-004 copyright rule).
- Keep prompts concise. Short fields read more naturally and avoid "machine-voiced" phrasing.
- Use [`reference-library.md`](reference-library.md) to validate workplace authenticity (COCA business register, ABEL word list).
- Every question must still pass the ADR-004 quality gate: style alignment, reference-library check, copy-editing sweep, and human sniff-test.
