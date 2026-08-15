# Question Schema

Canonical schema for TOEIC Writing practice questions. This document is the single source of truth for the JSON shape consumed by the app's `QuestionSelector` component (see ADR-003).

## Schema

```typescript
// Per-Part evaluation criteria — literal unions prevent typo'd criterion
// names from silently breaking ADR-002's evaluation schema
type RubricCriteriaPart1 = 'grammar' | 'task_completion' | 'word_usage';
type RubricCriteriaPart2 = 'sentence_quality_variety' | 'vocabulary' | 'organization';
type RubricCriteriaPart3 = 'thesis_and_support' | 'grammar' | 'vocabulary' | 'organization';

// Discriminated union: fields are required/optional based on Part
interface QuestionPart1 {
  part: 1;
  id: string;            // unique within Part 1, e.g. "p1-001"
  prompt: string;
  targetWords: string[];  // exactly 2 words/phrases the student must use
  scoringBand: { min: 0; max: 3 };
  rubricCriteria: RubricCriteriaPart1[];
  wordTarget?: { min: number; max: number; hint?: string };
  imageUrl?: string | null;
  imageAlt?: string;
}

interface QuestionPart2 {
  part: 2;
  id: string;            // unique within Part 2, e.g. "p2-001"
  prompt: string;
  emailContext: {        // the email the student responds to
    from: string;
    to: string;
    subject: string;
    sent: string;        // date/time string, e.g. "August 15, 10:30 A.M."
    body: string;
  };
  scoringBand: { min: 0; max: 4 };
  rubricCriteria: RubricCriteriaPart2[];
  wordTarget?: { min: number; max: number; hint?: string };
}

interface QuestionPart3 {
  part: 3;
  id: string;            // unique within Part 3, e.g. "p3-001"
  prompt: string;
  essayTopic?: string;    // optional — some questions may use prompt only
  scoringBand: { min: 0; max: 5 };
  rubricCriteria: RubricCriteriaPart3[];
  wordTarget?: { min: number; max: number; hint?: string };
}

type Question = QuestionPart1 | QuestionPart2 | QuestionPart3;
```

## rubricCriteria by Part

These map directly to ETS TOEIC Writing evaluation criteria (verified against the ETS Score User Guide, cited in ADR-002):

### Part 1 — Write a sentence based on a picture (Score 0–3)

| Criterion | Description |
| --- | --- |
| `grammar` | Appropriate and accurate use of grammar |
| `task_completion` | Sentence addresses the picture scenario and uses both required words/phrases |
| `word_usage` | Required words are integrated naturally (not forced) |

### Part 2 — Respond to a written request (Score 0–4)

| Criterion | Description |
| --- | --- |
| `sentence_quality_variety` | Range and accuracy of sentence structures |
| `vocabulary` | Appropriate word choice for a business context |
| `organization` | Logical flow and structure of the email |

### Part 3 — Write an opinion essay (Score 0–5)

| Criterion | Description |
| --- | --- |
| `thesis_and_support` | Clear opinion stated and supported with reasons/examples |
| `grammar` | Accurate and varied grammar throughout |
| `vocabulary` | Range and precision of vocabulary |
| `organization` | Logical paragraph structure and cohesive links |

## Scoring Band Reference

| Part | Task | Score Range |
| --- | --- | --- |
| Part 1 (Q1–5) | Write a sentence based on a picture | 0–3 |
| Part 2 (Q6–7) | Respond to a written request | 0–4 |
| Part 3 (Q8) | Write an opinion essay | 0–5 |

These ranges are fixed by ETS and must not be changed.

## Example Questions

### Part 1 — Sentence based on a picture

```json
{
  "part": 1,
  "id": "p1-001",
  "prompt": "Write ONE sentence based on a picture. Use the TWO words or phrases: 'conference' and 'reschedule'.",
  "targetWords": ["conference", "reschedule"],
  "scoringBand": { "min": 0, "max": 3 },
  "rubricCriteria": ["grammar", "task_completion", "word_usage"],
  "imageUrl": null,
  "imageAlt": "A conference room with a manager and two employees at a table with charts and a laptop."
}
```

### Part 2 — Email response

```json
{
  "part": 2,
  "id": "p2-001",
  "prompt": "You received the following email. Read it carefully, then write your response.",
  "emailContext": {
    "from": "Maria Chen, Event Coordinator",
    "to": "New Employee",
    "subject": "Welcome Orientation Details",
    "sent": "August 15, 9:15 A.M.",
    "body": "Welcome to TechFlow Solutions! Your orientation starts Monday at 9:00 A.M. in Conference Room B. Please bring a photo ID and your employment paperwork. We'll cover company policies, IT setup, and team introductions. Lunch will be provided. If you have any dietary restrictions, please let me know by Sunday evening."
  },
  "scoringBand": { "min": 0, "max": 4 },
  "rubricCriteria": ["sentence_quality_variety", "vocabulary", "organization"]
}
```

### Part 3 — Opinion essay

```json
{
  "part": 3,
  "id": "p3-001",
  "prompt": "Write an opinion essay. In your essay, be sure to state your opinion, develop your ideas with reasons and examples, and organize your response effectively.",
  "essayTopic": "Some people prefer to work from home full-time, while others prefer working in a traditional office. Which approach do you prefer? Use specific reasons and examples to support your opinion.",
  "scoringBand": { "min": 0, "max": 5 },
  "rubricCriteria": ["thesis_and_support", "grammar", "vocabulary", "organization"],
  "wordTarget": { "min": 250, "max": 350, "hint": "TOEIC Part 3 target: ~300 words" }
}
```

## Internal Reference: ETS Original Format (Do Not Embed)

These questions from the ETS TOEIC Writing sample test are used **only as style templates for human curators** — they are not embedded in the app's question bank:

- **Part 1**: Picture scenario + two target words (e.g., `airport terminal, so`)
- **Part 2**: Welcome email → respond with at least two requests for information
- **Part 3**: Job search methods → "What do you think is the best way to find a job?"

Curators use these ETS originals to calibrate naturalness and difficulty, then create new prompts with different scenarios.

## Versioning

- The JSON array root includes a `"schemaVersion": 1` field.
- Changes are **additive only** (new optional fields). Existing fields are never removed or repurposed.
- Major schema changes trigger a new ADR revision and a migration path in the import logic.

## Validation Rules

```typescript
// Pseudocode — enforced at build time by the author, and at import time for user data.
//
// Note: the calibration file (docs/questions-sample.json) uses non-conforming IDs
// (e.g. "ets-p1-01") and is intentionally excluded from this validator. It is
// internal reference only — real authored questions must follow the p${part}- convention.
function validateQuestion(q: Question): boolean {
  assert(q.part === 1 || q.part === 2 || q.part === 3);
  assert(q.id.startsWith(`p${q.part}-`));
  assert(q.scoringBand.min === 0);
  assert(q.scoringBand.max === { 1: 3, 2: 4, 3: 5 }[q.part]);
  assert(q.rubricCriteria.length > 0);
  if (q.part === 1) assert(q.targetWords.length === 2);
  if (q.part === 2) assert(q.emailContext != null);
  if (q.part === 3) assert(q.essayTopic && q.essayTopic.length > 10);
  return true;
}
```
