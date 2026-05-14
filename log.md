# Log

Chronological, append-only record of operations on this wiki. Maintained by the LLM agent.

Every entry starts with `## [YYYY-MM-DD] <op> | <title>` so you can quickly scan recent activity:

```bash
grep "^## \[" log.md | tail -5
```

Operations recorded: `ingest`, `query`, `lint`, `migration`.

---

_(no entries yet — drop a source into `raw/` or `wiki/_inbox/` and tell your agent to ingest)_
