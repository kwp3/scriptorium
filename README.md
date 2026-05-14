# LLM Wiki Starter

A starter template for an LLM-maintained personal wiki — incrementally compile your reading into a persistent, interconnected knowledge base instead of re-deriving everything on every query.

## What is this?

Most LLM-and-documents setups look like RAG: upload files, retrieve chunks at query time, generate an answer. The LLM rediscovers knowledge from scratch on every question. Nothing accumulates.

This template is different. The LLM **incrementally builds and maintains a persistent wiki** — a structured collection of interlinked markdown files that sits between you and the raw sources. When you add a source, the agent reads it, extracts the key information, and integrates it into the existing wiki: updating entity pages, revising concept pages, noting where new data contradicts old claims, strengthening the evolving synthesis. The cross-references are already there. The synthesis already reflects everything you've read. Knowledge is compiled once and kept current, not re-derived on every query.

You curate sources and ask questions. The agent does the bookkeeping that makes a knowledge base actually useful over time.

## Architecture

Three layers, by role:

- **`raw/`** — inputs. Unprocessed source material (articles, PDFs, transcripts, notes). Immutable once placed. Lives outside the Obsidian vault so it doesn't pollute the graph.
- **`wiki/`** — the knowledge product. Synthesized, interconnected, cited. This is the Obsidian vault root. Consumable by you OR any downstream agent — research, app-building, Q&A, slide generation, retrieval.
- **Project-root files** (`AGENTS.md`, `workflows/`, `log.md`) — pipeline metadata. How the wiki is built and what's happened during construction.

```
./
├── README.md                      # this file
├── AGENTS.md                      # operational spec — every agent reads this first
├── workflows/
│   ├── ingest.md                  # how to ingest a new source
│   ├── query.md                   # how to answer questions / generate outputs
│   └── lint.md                    # how to health-check the wiki
├── log.md                         # append-only chronological log
├── raw/                           # source files (inputs)
│   ├── articles/
│   ├── pdfs/
│   ├── transcripts/
│   └── assets/
└── wiki/                          # ← point Obsidian here
    ├── index.md
    ├── _tags.md
    ├── _inbox/                    # Web Clipper landing zone
    ├── sources/
    ├── entities/
    ├── concepts/
    ├── analyses/
    └── topics/
```

## Prerequisites

- **Obsidian** (https://obsidian.md) — for browsing the wiki. Optional but strongly recommended.
- **An LLM agent that can read this repo's markdown and edit files** — Claude Code, Codex, OpenCode, Continue, Cursor, etc. The spec is agent-agnostic.
- **Git** — every state-changing operation is committed.
- **Optional Obsidian plugins:** Dataview (for queries in `index.md` and lint), Marp (for slide-deck outputs).
- **Optional PDF extractor:** see `workflows/ingest.md` for the priority list. Any of `marker`, `docling`, `pymupdf`, or `pdftotext` works.
- **Optional search engine (recommended once the wiki grows past ~100 pages):** [qmd](https://github.com/tobi/qmd) — local hybrid BM25 + vector + LLM-reranking search. Install with `npm install -g @tobilu/qmd`. First `qmd embed` run downloads ~2–3GB of GGUF models (embedding, reranker, query-expansion) into `~/.cache/qmd/`.

## Getting started

1. **Clone this repo** and rename if you like (the spec doesn't depend on the folder name).
2. **Open the Obsidian vault** at `./wiki/`. Install Dataview and Marp if you want them.
3. **Configure Web Clipper** (optional) to write into `./wiki/_inbox/`.
4. **Set the Obsidian attachment folder path** to `../raw/assets/` so downloaded images land outside the vault.
5. **Run your LLM agent** with its working directory at the project root. It will read `AGENTS.md` and orient itself.
6. **Drop your first source** — a PDF in `raw/pdfs/`, an article via Web Clipper, a transcript in `raw/transcripts/`, or any markdown file at `raw/` root. Then say `ingest` to the agent.

After a few sources, ask a question — the agent will read the wiki, synthesize an answer with citations, and offer to file the result back as an analysis page.

## How to customize

The whole spec is in `AGENTS.md` plus the three files under `workflows/`. Edit those to fit your domain and workflow. Common customizations:

- Add or remove page types in the taxonomy (`AGENTS.md` → Page taxonomy section).
- Adjust the promotion thresholds for concepts and topics (`workflows/ingest.md` → Synthesis quality heuristics).
- Add domain-specific lint checks (`workflows/lint.md`).
- Add new output formats for queries (`workflows/query.md`).

The split between `AGENTS.md` (always loaded) and `workflows/*.md` (loaded on demand) keeps per-operation context cost low while giving each workflow room to be thorough.

## Credits

- **Andrej Karpathy's `llm-wiki` pattern** — the architectural inspiration. Original gist: https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f
- **Vannevar Bush's Memex** (1945) — the deeper conceptual ancestor. A personal, curated knowledge store where the associative trails between documents are as valuable as the documents themselves. Bush couldn't solve who does the maintenance. LLMs can. See "As We May Think" (https://en.wikipedia.org/wiki/As_We_May_Think).
- **qmd** by Tobi (https://github.com/tobi/qmd, npm: `@tobilu/qmd`) — optional local hybrid search engine, referenced in the query workflow.
