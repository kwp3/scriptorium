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
├── log.md                         # append-only operational log; newest at bottom
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
source_published_provenance: <free text>   # optional; include when the date comes from a fallback (e.g. "file mtime")
conversion_method: <tool name>             # omit entirely when not applicable; include only for PDFs / transcripts that required a conversion tool
tags:                                       # must include a source-type tag (#article, #pdf, #transcript, or #note) matching source_type above, plus topical tags
  - <source-type tag>
  - <topical tags...>
```
Required sections:
- `## Summary`
- `## Key claims` — bullets stating the source's claims. Do NOT self-cite (don't add `[[sources/THIS-PAGE]]`); claims on a source page implicitly derive from that source.
- `## Entities & concepts` — split into two required sub-sections:
  - `### Entities` — every person, org, product, place, or other named real-world thing the source meaningfully describes. Entities with pages get wikilinks; entities without pages yet are plain text. Each line requires a `(entity_kind)` annotation in parentheses — e.g. `**[[WikiProject AI Cleanup]]** (org)` or `**GPTZero** (product)`. Write `*(none)*` only when the source genuinely contains no named persons, orgs, products, or places with capturable facts.
  - `### Concepts` — promoted concepts as wikilinks, deferred concepts as plain text with source count (see promotion threshold in `workflows/ingest.md`). Promoted: `[[Concept Name]]` (page exists). Deferred: `Concept name (1 source — deferred)`.
- `## Raw` — **standard markdown link** to the file under `raw/`, e.g. `[<filename>](../../raw/<subfolder>/<filename>)`. Source pages live at `wiki/sources/`, so the relative path needs **two** `../` segments to reach the project root before descending into `raw/` — `../raw/` would resolve inside `wiki/` and 404. Do NOT use a wikilink with a relative path (`[[../../raw/...]]` doesn't resolve in Obsidian) and do NOT wrap the path in backticks (turns it into inline code, not a clickable link).

**Entity** (`wiki/entities/<canonical name>.md`) — people, organizations, products, places.
```yaml
type: entity
entity_kind: person | org | product | place | other
aliases: [...]   # different SURFACE FORMS of the same canonical thing (e.g. "V. Bush", "Bush, Vannevar" for "Vannevar Bush"). NOT distinct-but-related entities — e.g. "GPT-4" is NOT an alias for "ChatGPT"; they're separate entities (a model vs. a chatbot product). Do NOT include the canonical name itself (the page filename) as an alias — it is redundant.
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
aliases: [...]   # optional; alternate names for the same concept (e.g. "LLM Writing Patterns" for "LLM Stylistic Markers"). Used by de-duplication on future ingests — if a new source uses an alias, update the existing page rather than creating a duplicate. Do NOT include the canonical name itself as an alias.
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

**Mixed-format analyses.** Some analyses are produced in formats that have their own frontmatter or no frontmatter at all (Marp slide decks need `marp: true`/`theme:`/`paginate:`; Obsidian Canvas files (`.canvas`) are JSON and cannot carry YAML at all). These are still analyses, and the schema still applies — adapt as follows:

- **Marp slide decks** (`wiki/analyses/<title>.md` with Marp directives): keep the Marp keys AND include the analysis frontmatter alongside them in the same block. Place the required sections (`## Question`, `## Findings`, `## Caveats / gaps`) as a brief preface before the first `---` slide separator, then begin the deck. The preface anchors the deck inside the wiki's citation graph; the slides are the user-facing form.
- **Canvas analyses** (`wiki/analyses/<title>.canvas`): exempt from YAML frontmatter (the file is JSON). Instead, the catalog entry in `wiki/index.md` must include a fuller one-line summary that captures what would have been the `question:` field, and the canvas itself must include `file`-type nodes pointing to the sources it draws from (Obsidian renders these as live embeds and they serve as the citation graph).
- **Any other non-markdown output** filed under `analyses/` should follow the same principle: preserve as much of the analysis schema as the format allows, and capture the missing metadata in `wiki/index.md`.

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
- When naming an entity that has a wiki page in the body prose of a concept, analysis, or topic page, use `[[entities/<name>]]` — never `**bold**` or plain text. Bold without a wikilink is invisible to the link graph.

**Tags:**
- The controlled vocabulary lives in `wiki/_tags.md` — a flat list with a one-line gloss per tag.
- BEFORE applying a tag, read `wiki/_tags.md` and reuse an existing tag if a fit exists.
- New tags require adding a one-line entry to `wiki/_tags.md` in the same commit.
- Hierarchical tags are fine when sub-categorization is natural (`#topic/ai/safety`).
- Tags are for cross-cutting categorization (topic, source-type, importance), NOT for linking specific entities/concepts — use wikilinks for those.
- **Source-type tags are required on every source page.** Each page in `wiki/sources/` must carry exactly one of `#article`, `#pdf`, `#transcript`, or `#note`, matching its `source_type` frontmatter. These four tags are pre-registered in the template's `wiki/_tags.md`.
- **Format note for agents comparing the two.** In page frontmatter, tags are **bare strings** under the `tags:` key (e.g. `- article`). In `wiki/_tags.md`, the same tags are listed **backtick-wrapped** (e.g. `` `article` ``) purely for display readability — the backticks are not part of the tag. When comparing the frontmatter set against the registered set (e.g. in lint check 6), strip backticks from `_tags.md` before comparing.

## Indexing & logging

- `wiki/index.md` — content-oriented catalog, organized by page type. Each entry: wikilink + one-line summary. Updated on every state-changing operation.
- `log.md` — **append-only, newest entries at the BOTTOM.** New entries are *appended* (added to the end of the file), never prepended. Always insert by placing content after the final line of the file — do NOT use search-and-replace to position entries within the file, which risks inserting before existing content. Every entry starts with `## [YYYY-MM-DD] <op> | <title>` so `grep "^## \[" log.md | tail -5` returns the five most recent operations. Operations: `ingest`, `query`, `lint`, `migration`.

## Tooling references

Brief reference only; workflow files explain when and how to use each.

- **Obsidian Web Clipper** — browser extension; writes clipped articles into `wiki/_inbox/`.
- **Image download hotkey** (`Ctrl+Shift+D` by default) — Obsidian command "Download attachments for current file". Read text first, then view images separately; LLMs can't read markdown-with-inline-images in one pass.
- **Pluggable PDF extractor** — see `workflows/ingest.md` for the priority list.
- **qmd** (https://github.com/tobi/qmd) — optional local search engine; install with `npm install -g @tobilu/qmd`. Useful once the wiki grows beyond a few hundred pages.
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
