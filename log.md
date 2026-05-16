# Log

Chronological, append-only record of operations on this wiki. Maintained by the LLM agent.

Every entry starts with `## [YYYY-MM-DD] <op> | <title>` so you can quickly scan recent activity:

```bash
grep "^## \[" log.md | tail -5
```

Operations recorded: `ingest`, `query`, `lint`, `migration`.

---

_(no entries yet — drop a source into `raw/` or `wiki/_inbox/` and tell your agent to ingest)_

## [2026-05-15] migration | spec tightenings from test-wiki-5 post-ingest review
- files: AGENTS.md, workflows/ingest.md, workflows/lint.md, workflows/query.md
- changes:
  - workflows/ingest.md — clarified that entities have no source-count threshold; flagged empty entities folder as a near-certain misread
  - AGENTS.md — added "Mixed-format analyses" subsection covering Marp decks (preserve both frontmatters, preface with required sections) and Canvas (exempt from YAML, capture metadata in index.md)
  - workflows/lint.md — added Check 13 (suspiciously empty page-type folders, primary signal: sources >= 2 and entities == 0); bumped checks_run to 13
  - workflows/query.md — replaced "reusable synthesis" file-back trigger with a 4-bullet checklist; promoted unfiled-query logging from buried conditional to mandatory section with multi-question handling
- driver: test-wiki-5 ingest review surfaced four failure modes the prior spec did not prevent
