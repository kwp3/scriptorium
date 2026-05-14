# AGENTS.md

Operational spec for an LLM-maintained personal wiki. Any agent (Claude Code, Codex, OpenCode, etc.) operating in this repo reads this file first.

**To instantiate:** clone the repo, point your Obsidian vault at `./wiki/`, run an LLM agent with its working directory set to the project root, and drop sources into `./raw/` (or `./wiki/_inbox/` for Web Clipper). See `README.md` for the human-facing walkthrough.

## Philosophy

The wiki is a persistent, compounding synthesis layer between raw sources and the user. The agent owns maintenance (summarizing, cross-referencing, filing, bookkeeping); the user owns curation, direction, and the questions that matter. The wiki keeps getting richer with every source added and every question asked — knowledge is compiled once, then kept current, not re-derived on every query.

## Workflow router

**BEFORE acting on any of the triggers below, you MUST read the linked workflow file in full. Do not improvise from this file alone — the workflow files contain the operational detail.**

| Trigger | Workflow file |
| --- | --- |
| `ingest`, `process`, `add this source`, `process new sources`, OR any time a file appears in `raw/` or `wiki/_inbox/` | `workflows/ingest.md` |
| Any question about wiki content, request for synthesis, comparison, analysis, slide deck, chart, or canvas | `workflows/query.md` |
| `lint`, `health check`, `find issues`, periodic maintenance | `workflows/lint.md` |

If a request straddles workflows (e.g. "ingest these PDFs and then summarize what they all say about X"), read all relevant workflow files in order.

## Directory layout

The Obsidian vault root is `./wiki/`, NOT the project root. Everything outside `./wiki/` is agent-facing scaffolding and never appears in Obsidian's graph view or search.

```
./                                 # project root (agent CWD, git repo root)
├── README.md                      # onboarding for humans
├── AGENTS.md                      # this file — routing + universals
├── workflows/                     # workflow-specific specs
│   ├── ingest.md
│   ├── query.md
│   └── lint.md
├── log.md                         # append-only operational log
├── raw/                           # immutable source files
│   ├── *.md                       # generic markdown sources at root
│   ├── articles/                  # web clippings, after migration from wiki/_inbox/
│   ├── pdfs/                      # original PDFs + their *.converted.md siblings
│   ├── transcripts/               # podcast / video transcripts
│   └── assets/                    # downloaded images (Obsidian Attachment folder)
└── wiki/                          # ← Obsidian vault root points HERE
    ├── index.md                   # browsable catalog of all wiki pages
    ├── _tags.md                   # controlled tag vocabulary
    ├── _inbox/                    # Web Clipper landing zone; drained on ingest
    ├── sources/                   # one summary per ingested source
    ├── entities/                  # people, orgs, products, places
    ├── concepts/                  # ideas, theories, frameworks
    ├── analyses/                  # comparisons, syntheses, query-derived insights
    └── topics/                    # theme overviews stitching the above
```

**Three-layer routing rule (by role, not by audience):**

- `raw/` — **inputs.** Unprocessed source material. Immutable once placed. Not knowledge yet.
- `wiki/` — **the knowledge product.** Synthesized, interconnected, cited. Open for any downstream use by anyone (you, agents, other tools) — research, app-building, Q&A, slide generation, training data, retrieval. No audience constraint.
- Project-root files (`AGENTS.md`, `log.md`, `workflows/`) — **pipeline metadata.** Configuration for and telemetry about how the wiki gets built, not knowledge about the world.

Web Clipper writes inside the vault as a quirk of the extension, so `wiki/_inbox/` is a transient landing zone (not a fourth layer) — drained on every ingest. See `workflows/ingest.md`.

## Page taxonomy + frontmatter schemas

All pages use YAML frontmatter so Dataview can query them. Required fields are listed; pages may add others.

**Source summary** (`wiki/sources/<title>.md`) — one per ingested source.
```yaml
type: source
created: YYYY-MM-DD
ingested: YYYY-MM-DD
source_type: article | pdf | transcript | note | other
source_url: <url or null>
source_published: YYYY-MM-DD | unknown
conversion_method: <tool name, for PDFs/transcripts>
tags: [...]
```
Required sections: `## Summary`, `## Key claims`, `## Entities & concepts` (wikilinks to pages this source touched), `## Raw` (relative link back to the file under `raw/`).

**Entity** (`wiki/entities/<canonical name>.md`) — people, organizations, products, places.
```yaml
type: entity
entity_kind: person | org | product | place | other
aliases: [...]
created: YYYY-MM-DD
updated: YYYY-MM-DD
sources: ["[[source-title-1]]", ...]
tags: [...]
```
Required sections: `## Overview` (1–3 sentences), `## Key facts` (each cited via wikilink to a source), `## Related` (wikilinks to other entities/concepts).

**Concept** (`wiki/concepts/<name>.md`) — ideas, theories, frameworks.
```yaml
type: concept
created: YYYY-MM-DD
updated: YYYY-MM-DD
sources: ["[[source-title-1]]", ...]
tags: [...]
```
Required sections: `## Definition`, `## Origins / proponents`, `## Examples`, `## Critiques or contradictions`.

**Analysis** (`wiki/analyses/<title>.md`) — comparisons, syntheses, query-derived insights filed back from the user.
```yaml
type: analysis
created: YYYY-MM-DD
question: <the question this analysis answers>
sources: ["[[source-title-1]]", ...]
tags: [...]
```
Required sections: `## Question`, `## Findings`, `## Caveats / gaps`.

**Topic / theme overview** (`wiki/topics/<theme>.md`) — high-level rollups.
```yaml
type: topic
created: YYYY-MM-DD
updated: YYYY-MM-DD
entities: ["[[entity-1]]", ...]
concepts: ["[[concept-1]]", ...]
tags: [...]
```
Required sections: `## Overview`, `## Key entities`, `## Key concepts`, `## Open questions`.

## Wikilink and tag conventions

**Wikilinks:**
- Use `[[Wikilink]]` syntax for ALL inter-wiki references — never markdown links.
- Page filenames mirror display titles (Obsidian-style).
- Backlinks are implicit via Obsidian; do not maintain a manual "linked from" list.
- Cite sources via `[[sources/<title>]]` wherever a non-trivial claim appears.

**Tags:**
- The controlled vocabulary lives in `wiki/_tags.md` — a flat list with a one-line gloss per tag.
- BEFORE applying a tag, read `wiki/_tags.md` and reuse an existing tag if a fit exists.
- New tags require adding a one-line entry to `wiki/_tags.md` in the same commit.
- Hierarchical tags are fine when sub-categorization is natural (`#topic/ai/safety`).
- Tags are for cross-cutting categorization (topic, source-type, importance), NOT for linking specific entities/concepts — use wikilinks for those.

## Indexing & logging

- `wiki/index.md` — content-oriented catalog, organized by page type. Each entry: wikilink + one-line summary. Updated on every state-changing operation.
- `log.md` — chronological, append-only. Every entry starts with `## [YYYY-MM-DD] <op> | <title>` so `grep "^## \[" log.md | tail -5` works. Operations: `ingest`, `query`, `lint`, `migration`.

## Tooling references

Brief reference only; workflow files explain when and how to use each.

- **Obsidian Web Clipper** — browser extension; writes clipped articles into `wiki/_inbox/`.
- **Image download hotkey** (`Ctrl+Shift+D` by default) — Obsidian command "Download attachments for current file". Read text first, then view images separately; LLMs can't read markdown-with-inline-images in one pass.
- **Pluggable PDF extractor** — see `workflows/ingest.md` for the priority list.
- **qmd** (https://github.com/tobi/qmd) — optional local search engine for the wiki; useful once the wiki grows beyond a few hundred pages.
- **Dataview** — Obsidian plugin; used by `workflows/lint.md` queries and for dynamic indexes inside `wiki/index.md`.
- **Marp** — Obsidian plugin; used by `workflows/query.md` for slide-deck outputs.
- **Git** — commit after every state-changing operation; one ingest = one commit.

## General agent conventions

- Never modify anything under `raw/` after it's been placed (immutable source of truth). Routing moves via `git mv` happen ONCE on first touch — see `workflows/ingest.md`.
- `wiki/_inbox/` is transient — drain it on ingest; never write summaries into it.
- Update `wiki/index.md` and `log.md` on every state-changing operation.
- Prefer editing existing pages over creating duplicates. Search by entity name (and aliases) first.
- Cite sources via wikilinks back to `[[sources/<title>]]` pages.
- Use Obsidian-flavored markdown (callouts, frontmatter, embeds) where it adds value.
- Commit message format: `<op>: <title>` (e.g. `ingest: As We May Think`).
- When uncertain about a routing or schema choice, ask the user rather than guessing.
