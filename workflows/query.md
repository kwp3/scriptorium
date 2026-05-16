# workflows/query.md

Query workflow. Read this in full before answering questions, generating syntheses, or producing outputs from the wiki. Universals live in `AGENTS.md`.

## Entry point

This workflow applies when the user asks a question, requests a synthesis, comparison, deck, chart, or canvas based on wiki content — anything beyond reading a single page. Examples:

- "What do my sources say about X?"
- "Compare Y and Z."
- "Make me a slide deck about W."
- "Chart the timeline of A."
- "What's the relationship between B and C across what I've read?"

## Retrieval

1. **Read `wiki/index.md` first.** It's the catalog organized by page type — fastest way to identify candidate pages.
2. **Drill into relevant pages.** Read source summaries, entity / concept pages, and existing analyses that touch the question.
3. **For large wikis (>100 pages):** if `qmd` is installed (`npm install -g @tobilu/qmd`), shell out for hybrid BM25 + vector + LLM-reranking search. Add the wiki as a collection once (`qmd collection add ./wiki --name <name>` then `qmd embed`), and query with `qmd query "<question>"`. Otherwise the `wiki/index.md` catalog is sufficient. Note: `qmd embed` downloads ~2–3GB of GGUF models on first run.
4. **Follow wikilinks** to expand context. A good answer often emerges from following 2–3 hops out from the seed pages.
5. **Note source dates.** If `source_published` is old and the topic is fast-moving, flag the staleness in the answer.

## Answering

- Answer with **explicit citations** — every claim links back to its `[[sources/<title>]]` source-summary page. The user should be able to click through to verify.
- If sources disagree, surface it. Don't paper over contradictions; either reconcile explicitly or report the disagreement.
- If the wiki has insufficient coverage to answer well, say so. Suggest sources to ingest next.
- Distinguish what the sources say from what you're synthesizing on top of them. Mark synthesis explicitly ("synthesizing across the sources: ...").

## Output formats

Pick the format that best fits the request. If unclear, ask.

### Markdown page (default)
A direct prose answer with citations and section headers. Good for most questions.

### Marp slide deck
A `.md` file with Marp frontmatter, suitable for the Marp Obsidian plugin. Use when the user asks for slides, a deck, or a presentation.
```markdown
---
marp: true
theme: default
---
# Title
- bullet
---
# Slide 2
```
Save to `wiki/analyses/<title>.md` if filed back (see below), or output inline if ephemeral.

### Matplotlib chart
A Python snippet that produces a chart. Save the script alongside the chart so it's reproducible. Output PNG/SVG to `raw/assets/` (NOT `wiki/`, since charts are derived artifacts and the script is the source of truth).

### JSON Canvas
A `.canvas` file with nodes and edges, suitable for Obsidian's Canvas view. Use when the user asks for a visual map, spatial layout, or mind-map of relationships.

## File-back pattern (important)

When the answer represents reusable synthesis, **offer to file it back** as a new analysis page:

> "This synthesis seems reusable. Want me to file it as `wiki/analyses/<title>.md` so future queries can build on it?"

Do NOT auto-file. The user decides what's keepable. But the offer is **not** optional — agents tend to skip it on long prose answers, which is exactly when filing matters most. **Offer to file if any of these are true:**

- The answer is a **comparison** across 2+ sources (e.g. "compare X and Y", "how do these sources differ on Z").
- The prose exceeds **~6 substantive paragraphs** or weaves together 3+ sources of synthesis.
- The output is a **structured artifact** — slide deck, JSON Canvas, chart, table that integrates multiple sources.
- The user asks a question whose answer composes facts from sources in a way the wiki's existing pages don't already capture (i.e. it's genuinely new synthesis, not a lookup).

Pure lookups ("what does my wiki say about X?"), gap reports ("we have no source on Y"), and one-paragraph clarifications don't need the offer. Everything else does. When in doubt, offer — the cost of asking is trivial; the cost of losing a substantive synthesis is permanent.

If filed:

1. Write to `wiki/analyses/<title>.md` using the analysis schema from `AGENTS.md` (`type: analysis`, `question:`, `sources: [...]`). For mixed-format outputs (Marp decks, Canvas), follow the "Mixed-format analyses" subsection in `AGENTS.md`.
2. Add a one-line entry to `wiki/index.md` under the Analyses section.
3. If new tags were used, add them to `wiki/_tags.md`.
4. Append a `log.md` entry:
   ```
   ## [YYYY-MM-DD] query | <title>
   - question: <the user's question>
   - filed: yes
   - sources: <N referenced>
   ```
5. Commit: `git commit -m "query: <title>"`.

## Logging unfiled queries (mandatory)

**Every query gets a log entry, regardless of file-back outcome.** Agents routinely skip this step for unfiled queries, leaving the wiki with no trail of what has been explored — which means recurring questions get re-derived from scratch and the user can't see at a glance what they've already asked. Don't skip it.

If the user declined to file (or the answer didn't warrant an offer), append a lighter entry **without committing**:

```
## [YYYY-MM-DD] query | <short description of the question>
- filed: no
- sources: <N referenced>
```

No commit on unfiled queries — the log entry is the only state change, and it'll go in with the next ingest or lint commit. For multi-question sessions, log each question as a separate entry; do NOT collapse them into one session-level entry, since each question is independently searchable for future "have we explored this?" checks.

This trail is what lets future agents (and the user) see what has already been explored, spot recurring questions that should be turned into evergreen analyses, and avoid duplicate work.
