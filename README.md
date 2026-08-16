# English Active BYOK

A browser-based TOEIC Writing practice tool. Write responses to Part 1 (sentence from picture), Part 2 (email reply), and Part 3 (opinion essay), then get AI-powered feedback on grammar, vocabulary, organization, and other ETS rubric criteria.

## How it works

Users bring their own Google Gemini API key — paste it directly into the browser, and it never leaves your machine. No backend, no accounts, no telemetry. Everything runs as a single static page deployed to GitHub Pages.

## Project structure

```text
english-active-byok/
├── index.html                # Static app shell (HTML + inline scripts)
├── docs/                     # All architecture documentation
│   ├── decisions/            # Architecture Decision Records (ADRs)
│   ├── technical-notes/      # Implementation companions for each ADR
│   ├── *.md                  # Reference guides (models, schema, sources)
│   └── questions-sample.json # Internal calibration (gitignored)
└── .gitignore
```

See [`docs/README.md`](docs/README.md) for documentation navigation.

## Status

The current `index.html` is a working prototype with 3 practice questions. The full question bank, model selection dropdown, and multi-file component architecture are documented in `docs/decisions/` but not yet implemented.
