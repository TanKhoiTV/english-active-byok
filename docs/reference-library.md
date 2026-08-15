# Reference Library for Question Authors

A curated list of open, reliable sources for authentic TOEIC Writing question content and vocabulary. Authors use these to inspire topics, validate vocabulary, and maintain natural tone — **never to copy directly**. See ADR-004 for governance.

## Format Templates (Internal Calibration)

| Source | What | License | Internal use? |
| --- | --- | --- | --- |
| ETS TOEIC Writing Sample Test (ets.org) | Official format, 1 sample per Part | © ETS — sample practice only | ✅ Yes — 3 questions in `docs/questions-sample.json` |
| TestSUCCEED TOEIC Writing Samples | 7 samples (4 Part 1, 2 Part 2, 1 Part 3) | Educational (unclear) | ✅ Yes — additional 7 in `docs/questions-sample.json` |
| Estudyme TOEIC Writing Guide | 8 sample questions (5/2/1) with scoring explanations | Copyrighted (adapted/original) | ✅ Yes, for scenario reference |
| Jaime Selwood Practice Test (2018 PDF) | 8 questions (5/2/1) | Educational blog (unclear) | ✅ Yes, as scenario inspiration |

## ETS Scope: No Formal Syllabus (Format Defines Topics)

ETS does **not** publish a topic syllabus. Per the official *[TOEIC Speaking & Writing Sample Tests PDF](https://www.ets.org/pdfs/toeic/toeic-speaking-writing-sample-tests.pdf)* (p. 8):

> *"The TOEIC Speaking and Writing tests are not based on the content of any particular English course but, rather, on your proficiency."*
> *"The tests assess English-language speaking and writing proficiency and do not require candidates to have specialized knowledge of business."*
> *"Set in contexts appropriate for daily life and the global workplace... tasks that people might perform in work-related situations or in familiar daily activities that are common across cultures."*

**The "scope" is defined by the format itself** — three task types with specific evaluation criteria:

### Part 1 — Write a Sentence Based on a Picture

- **ETS task type**: Q1–5, write ONE sentence using TWO given words/phrases
- **ETS evaluation**: Grammar, Relevance of sentence to the picture
- **Internal reference**: `docs/questions-sample.json` — ETS sample prompt: *"airport terminal, so"*
- **Time**: 8 minutes total for all 5 questions

### Part 2 — Respond to a Written Request

- **ETS task type**: Q6–7, respond to an e-mail (e.g., welcome letter asking for ≥2 requests)
- **ETS evaluation**: Quality and variety of sentences, Vocabulary, Organization
- **Internal reference**: `docs/questions-sample.json` — ETS sample: Dale City Welcome Committee e-mail
- **Time**: 10 minutes each

### Part 3 — Write an Opinion Essay

- **ETS task type**: Q8, state, explain, and support opinion on a workplace issue
- **ETS evaluation**: Thesis & support, Grammar, Vocabulary, Organization
- **Internal reference**: `docs/questions-sample.json` — ETS sample topic: "What is the best way to find a job?"
- **Time**: 30 minutes, minimum 300 words

### What This Means for Authors

Since ETS does not prescribe topics, the **question topics** are our discretion — but they must fall within the "daily life and global workplace" context. The **evaluation criteria** are fixed by ETS and are already aligned in our schema (`docs/question-schema.md`). The **vocabulary level** is "everyday life and work activities" — not specialized jargon. Use the Business Vocabulary sources below to validate that your prompts stay within this band.

## Workplace Scenarios (Topic Inspiration)

| Source | What | License | Notes |
| --- | --- | --- | --- |
| KPU Business Communications Cases | 2 real workplace email scenarios (Sprinklez Donuts virus update, etc.) with error-correction examples | CC-BY-NC 4.0 | Gold standard for authentic business email tone and common mistakes |
| TestSUCCEED TOEIC Writing Samples | 7 questions (4/2/1) with email/image prompts | Educational (unclear) | Scenario variety: sports club, lease dispute, etc. |
| BusyTeacher.org | 1000+ business English worksheet topics (email complaints, meeting requests, etc.) | Free registration | Good for Part 2 email scenario ideation |

## Business Vocabulary (Authenticity Validation)

| Source | What | License | Notes |
| --- | --- | --- | --- |
| Academic Business English List (ABEL) | 840 word families for business English | Academic research (Concordia) | Cross-reference Part 2/3 vocabulary for professional register |
| Wolverhampton Business English Corpus (WBE) | Business English corpus from Commerce faculty texts | ELRA-W0028 (academic license) | Academic/business register word frequency data |
| COCA Business Register | 550K+ business English words from contemporary American English | Free account required | Search: "What does [word] mean?" → filter by "Business" register |
| Corpus of Contemporary American English (COCA) | 1 billion word corpus, business subsection | Free account | Validates that vocabulary appears in real business contexts, not just textbooks |

## Authoring Workflow

### Step 1: Topic Brainstorm

1. Pick a Part (1, 2, or 3).
2. Browse **KPU cases** or **BusyTeacher** for 3–5 scenario ideas (e.g., "conference room scheduling conflict", "supplier complaint about damaged goods").
3. For each scenario, ask: "Would a TOEIC test-taker recognize this as a real workplace situation?"

### Step 2: Vocabulary Check

1. Identify 5–10 key vocabulary words in your prompt (for Part 2/3) or target words (for Part 1).
2. Cross-reference each in **COCA** (business register) or **ABEL**.
3. If a word doesn't appear in business contexts, replace it with a more authentic term.

### Step 3: Write Original Prompt

1. Follow the ETS format exactly (see format templates above).
2. Replace ETS scenarios with your original ones from the brainstorm.
3. Ensure the `rubricCriteria` field matches the ETS sub-criteria (see `docs/question-schema.md`).

### Step 4: Quality Sweeps

1. **Copy-editing skill** (`.pi/skills/copy-editing/`): Run Clarity, Voice & Tone, and Specificity sweeps.
2. **Human sniff-test**: "Does this sound like it could be on the real TOEIC?"
3. **Register check**: Is the tone professional but accessible? (Use KPU cases as reference.)

### Step 5: Add to Bank

1. Validate against `docs/question-schema.md` schema.
2. Add to `questions.json` (Phase 2) or inline HTML script tag (Phase 1).
3. Update the question count in ADR-004's Phase 1/Phase 2 trigger thresholds.

## Example Transformation Chain

```text
KPU case concept: "Employee emails supervisor about conference room booking conflict"
→ Brainstorm: "Meeting room double-booked, need to reschedule"
→ COCA vocab check: "conference", "schedule", "reschedule" all appear in business register ✓
→ Original Part 1 prompt: "Write ONE sentence based on a picture using 'conference' and 'reschedule'"
→ Image description: "A conference room with a manager and two employees at a table with charts"
→ rubricCriteria: ["grammar", "task_completion", "word_usage"]
```

## Future Sources (Watch List)

| Source | Potential use | Access level |
| --- | --- | --- |
| ELRA-W0028 (WBE Corpus) | Large-scale business vocabulary frequency data | Request academic license |
| iTalki/HelloTalk community posts | Authentic business email language from ESL learners | Public browsing |
| Reddit r/businessEnglish | Real workplace communication examples | Public browsing |

## Notes

- All sources listed are for **authoring reference only**. Question text in the app must always be original.
- The ETS PDF and test-prep site questions are **internal calibration only** — never embedded in the public app.
- KPU cases (CC-BY-NC) are the closest thing to a "safe to reference" source for workplace email tone and common phrasing patterns, but even these should be used for inspiration, not direct copying.
