# workflows/ingest.md

Detailed ingest workflow. Read this in full before processing any source. Universals (page schemas, taxonomy, conventions) live in `AGENTS.md` — don't redefine them here, reference them.

## Entry point

The user invokes ingest by saying any of: `ingest`, `ingest new sources`, `process`, `add this source`, `ingest <filename>`. Files appearing in `raw/` or `wiki/_inbox/` are an implicit trigger.

### Pending-source detection (when invoked without a filename)

Goal: find every file in `raw/` and `wiki/_inbox/` that hasn't been ingested yet.

The canonical "has been ingested" marker is a corresponding `wiki/sources/<title>.md` page whose `## Raw` link points back to the raw file. Filename stems alone are unreliable — a paper at `raw/pdfs/2604.11687v1.pdf` becomes a source page titled `Please Make it Sound like Human`, so don't try to match titles to filenames. Cross-check via the `## Raw` link.

1. **List candidate raw files.** Every file under `raw/` (root + subfolders, excluding `assets/` and `*.converted.md` derivatives) and `wiki/_inbox/`.
2. **For each candidate**, grep `wiki/sources/*.md` for the raw filename. A match means already ingested; no match means pending.
3. **Sanity-check against `log.md`.** Every pending file should also lack a `## [YYYY-MM-DD] ingest | ...` entry that references it. If the log has an entry but `wiki/sources/` doesn't have a matching page (or vice versa), the previous ingest was incomplete — flag this for the user before starting a new one.
4. **Present the pending list.** If exactly one source is pending, announce it and proceed. If multiple are pending, or any title looks ambiguous, confirm with the user before starting.
5. Process one source at a time. **One ingest = one commit.**

**Bash one-liner** for steps 1–2 (Linux / macOS / Git Bash):
```bash
for f in $(find raw/ wiki/_inbox/ -type f \! -name '.gitkeep' \! -name '*.converted.md' 2>/dev/null); do
  grep -lF "$(basename "$f")" wiki/sources/*.md >/dev/null 2>&1 || echo "pending: $f"
done
```

### Re-ingestion handling

If a `wiki/sources/<title>.md` page already exists for the file:
- Ask the user: **skip** (default), **replace** (overwrite summary, re-integrate downstream pages), or **version** (keep old page, create `<title>-v2`).
- Never silently re-process. Stale summaries are less harmful than overwriting curated synthesis without consent.

## Auto-routing

When a file appears outside its canonical subfolder, route it on first touch using `git mv` (preserves history). Log the move as part of the ingest entry.

| File pattern | Destination |
| --- | --- |
| `*.pdf` | `raw/pdfs/` |
| `*.txt`, `*.vtt`, `*.srt` | `raw/transcripts/` |
| `*.md` with Web Clipper frontmatter (e.g. `source:` containing a URL, or `created:` with a clip timestamp) | `raw/articles/` |
| `*.md` other | stays at `raw/` root (generic) |
| Anything in `wiki/_inbox/` | `raw/articles/` (Web Clipper assumption) — sniff frontmatter to confirm |

If the type is ambiguous (e.g. a `.md` with no frontmatter that could be a clipped article or a hand-written note), ask the user.

## Sub-workflows by source type

### a) Web articles

1. Clipping lands in `wiki/_inbox/`.
2. **Move** to `raw/articles/` via `git mv` (step 1 of every article ingest). Lint flags any leftover in `wiki/_inbox/` as an error state.
3. If the article references images: invoke Obsidian's `Download attachments for current file` (default `Ctrl+Shift+D`) so images land in `raw/assets/`. Read the article text first, then optionally view downloaded images for context.
4. Proceed to common end-of-ingest steps below.

> [!warning] Frontmatter pitfall — do not mimic the Web Clipper's frontmatter on the source page
> Web Clipper writes its own YAML frontmatter into the clipped `.md` (typically `title:`, `source:` or `url:`, `author:` / `authors:`, `published:` or `date:`, sometimes `tags:`). That frontmatter belongs to the **raw file**, not your source-summary page. Under context pressure (long articles, multiple compactions), the easy failure mode is to copy the clipper's frontmatter onto `wiki/sources/<title>.md` and then add a `## Summary` underneath. **Don't.** The source-summary schema in `AGENTS.md` is strict — rewrite the frontmatter, do not preserve the clipper's.
>
> Mapping from typical Web Clipper fields → source-summary schema:
>
> | Web Clipper field | Source-summary field | Notes |
> | --- | --- | --- |
> | `title:` | (none — page filename is the title) | drop |
> | `source:` or `url:` | `source_url:` | rename |
> | `author:` / `authors:` | (none — capture authorship in the body if material) | drop from frontmatter |
> | `published:` or `date:` | `source_published:` | rename; if absent, fall back to file mtime and add `source_published_provenance: file mtime` |
> | `created:` (clip timestamp) | `ingested:` | the clip date IS the ingest date if you process it immediately; otherwise use today |
> | (always required) | `type: source` | always add |
> | (always required) | `created:` | today's date |
> | (always required) | `source_type:` | `article` for clipped content |
> | (always required) | `tags:` | must include `article` plus topical tags |
>
> Before-and-after example:
>
> ```yaml
> # ❌ BEFORE — Web Clipper frontmatter pasted into wiki/sources/<title>.md
> ---
> title: Wikipedia: Signs of AI Writing
> source: https://en.wikipedia.org/wiki/Wikipedia:Signs_of_AI_writing
> author: WikiProject AI Cleanup
> date: 2025-11-20
> created: 2026-05-14
> tags: [wikipedia, ai-writing]
> ---
> ```
>
> ```yaml
> # ✅ AFTER — rewritten to source-summary schema
> ---
> type: source
> created: 2026-05-14
> ingested: 2026-05-14
> source_type: article
> source_url: https://en.wikipedia.org/wiki/Wikipedia:Signs_of_AI_writing
> source_published: 2025-11-20
> tags:
>   - article
>   - wikipedia
>   - ai-writing-detection
> ---
> ```
>
> If you find yourself preserving the clipper's frontmatter shape unchanged, stop and rewrite. Lint check 12 (frontmatter schema validation) will catch this after the fact, but catching it during the ingest costs ~30 seconds and a follow-up commit.

### b) PDFs (seamless workflow — the priority pain point)

1. PDF lands in `raw/pdfs/<title>.pdf` (route from `raw/` root if it landed loose).
2. Extract text and tables using the **highest-quality available** PDF tool. Priority order:
   - `anthropic-skills:pdf` skill (best, if running Claude Code with skills available)
   - `marker` (https://github.com/datalab-to/marker) — best open-source PDF→markdown for academic papers
   - `docling` (https://github.com/docling-project/docling) — IBM, strong layout preservation
   - `pymupdf` (`fitz`) — fast, programmatic, no OCR
   - `pdftotext` (poppler) — last resort, plain text only

   **Tables are `pdftotext`'s primary failure mode.** If only `pdftotext` is available and the PDF contains structured tables, `pdftotext` will mangle them — numeric values scatter across single-column lines with no row/column structure, making any quantitative claim from the table unreliable. Before proceeding, prefer to install `pymupdf` (`pip install pymupdf` — fast, no system dependencies, no OCR but handles table layout reasonably). If installing isn't possible, **warn the user explicitly** that table content from this PDF may be inaccurate, then proceed with `pdftotext`.
3. Write converted markdown to `raw/pdfs/<title>.converted.md` so it's diffable, re-readable, and doesn't require re-OCR on subsequent reads.
4. Record the extractor used in the source summary's frontmatter (`conversion_method: marker`, etc.) so quality issues are traceable.
5. If the PDF contains figures/tables critical to the content and the extractor missed them: extract images separately into `raw/assets/` and view them inline as part of the summary step.
6. Proceed to common end-of-ingest steps below, reading from `<title>.converted.md` (not the original PDF).

### c) Transcripts (podcast / video)

1. Transcript lands in `raw/transcripts/<title>.{txt,vtt,srt,md}`.
2. Strip speaker noise, normalize timestamps to `[HH:MM:SS]` if present, segment by topic if the source is long.
3. Proceed to common end-of-ingest steps below.

### d) Generic markdown (raw/ root)

1. Any loose `.md` at `raw/` root is a misc source — notes, exports, briefs, book notes, etc.
2. Read as-is. Infer the source type from content and record it in the source summary's frontmatter (`source_type: note | brief | export | book-notes | other`).
3. No conversion needed.
4. Proceed to common end-of-ingest steps below.

## Synthesis quality heuristics

Apply these during the summarize-and-integrate phase of every ingest:

- **Cite every claim.** Any non-trivial assertion in an entity / concept / analysis / topic page gets a `[[sources/<title>]]` wikilink to its source. Unsupported claims get removed or flagged inline.
  - **Exception — source pages.** Claims on a source page implicitly derive from that source. Do NOT self-cite (don't add `[[sources/THIS-PAGE]]` to bullets on the source page itself). Citations exist to disambiguate where claims came from on pages that span multiple sources — that's not relevant on a page with exactly one source by definition.
- **De-duplicate entities and concepts before creating.** Before creating a new entity or concept page, search existing pages for fuzzy matches (`Vannevar Bush` vs `V. Bush` vs `Bush, Vannevar`; `LLM Writing Patterns` vs `LLM Stylistic Markers`). If a match exists, update it; add the surface form to the `aliases:` frontmatter field. Establish ONE canonical name and use it everywhere.
  - **Aliases are surface forms of the same thing**, not distinct-but-related items. `GPT-4` is NOT an alias for `ChatGPT` — they're separate entities (a model vs. a chatbot product) that happen to be related.
- **Surface contradictions explicitly.** When a new source contradicts an existing claim on an entity / concept page, add an Obsidian callout:
  ```markdown
  > [!conflict] Sources disagree
  > [[sources/A]] says X. [[sources/B]] says Y. Reconciliation: <your synthesis or "unresolved">.
  ```
  Never silently overwrite.
- **Date the sources.** Always populate `source_published` in source-summary frontmatter when known. This is what lint uses to detect stale claims.
- **Promotion thresholds.**
  - A passing mention stays inline inside its source summary.
  - **Entities have no source-count threshold.** Create an entity page (person, org, product, place) the first time a source meaningfully describes it. The "passing mention stays inline" rule still applies to truly incidental references — a single name-drop with no facts attached, e.g. "as IBM noted" used purely as throat-clearing — but anything with two or more facts worth capturing gets a page on first appearance. This explicitly differs from concepts: do NOT apply the 2-source threshold to entities. A wiki with multiple sources and zero entity pages is almost always wrong — the agent has misread this rule and deferred entities that should have been created. If unsure between "passing mention" and "first substantive mention," err on the side of creating the page; thinly populated entity pages are cheap and lint will flag the truly empty ones.
  - A concept gets its own `wiki/concepts/` page once **2+ sources** reference it.
  - A topic / theme overview gets created once **3+ concepts** cluster around it.
- **No new info without a source.** Never fabricate connections or claims. Every assertion in `wiki/` traces back to a file under `raw/`.
- **Prefer updates over creation.** If an entity / concept page already exists, update it. Only create when no match exists.
- **Deferred concepts are PLAIN TEXT, not wikilinks.** A concept that hasn't met the 2-source promotion threshold gets mentioned as plain text in the source's `## Entities & concepts` section and anywhere else in prose — e.g. `... see Diffusion Model Prompting for more (1 source so far, page deferred)`. Do NOT write `[[concepts/Diffusion Model Prompting]]` while the page doesn't exist; Obsidian will show it as a broken link and lint will flag it. Promote to wikilink only when you create the page.
- **Cross-link new and updated concept pages.** When you create or update a `wiki/concepts/<name>.md` page, add `[[concepts/<name>]]` to the `## Related` section of every entity referenced on that concept page. Backlinks are implicit in Obsidian's graph, but a visible Related section helps readers and lint.

## Entity & concept inventory (mandatory pre-step)

Before writing any pages, produce an entity & concept inventory in the chat. This is your synthesis plan — classify everything before creating anything. Do not skip this step under time pressure.

```
=== Entity & concept inventory: [source title] ===

Persons:  [names or "(none)"]
Orgs:     [names or "(none)"]
Products: [names or "(none)"]
Places:   [names or "(none)"]
Other:    [names or "(none)"]

→ Entity pages to create: [list with entity_kind in parens, e.g. "BART (product)"]
→ Entity pages to update: [list or "(none)"]
→ Passing mentions only (no page): [list or "(none)"]

Concept candidates:
  [concept name] — [N] source(s) [(deferred | promote)]

→ Concepts to promote: [list or "(none)"]
→ Concepts to defer:   [list or "(none)"]
```

Rules:
- A named thing goes in the entity list if it has two or more capturable facts (what it does, who made it, a key finding about it). A single name-drop with no attached facts stays off the list. When in doubt, list it — thinly populated entity pages are cheap.
- Every item in the entity list gets a page — no source-count threshold applies to entities.
- Concept candidates are held to the 2-source threshold, independent of the entity list.
- If unsure whether something is an entity or a concept: does it have a proper name and a persistent real-world existence? If yes, it's an entity.

The inventory is not committed to disk. Its output is the `### Entities` / `### Concepts` sub-sections of the source page (which are saved and lintable).

## Common end-of-ingest checklist

For every source, in order:

1. Write the source summary page in `wiki/sources/<title>.md` using the source-summary schema from `AGENTS.md`.
2. For each entity / concept / topic mentioned: update the existing page (preferred) or create a new one. Apply the synthesis quality heuristics.
3. Update `wiki/index.md` with new pages and updated one-liners.
4. If new tags were introduced: add them to `wiki/_tags.md` with a one-line gloss.
5. **Append (do not prepend)** an entry to the END of `log.md`. The log is sorted oldest-at-top, newest-at-bottom, so `tail -5` returns the five most recent operations. New entries go at the bottom of the file, after every existing entry. Format:
   ```
   ## [YYYY-MM-DD] ingest | <title>
   - source_type: <type>
   - touched: <N pages>
   - new: <entity/concept/topic page names>
   - notes: <anything noteworthy — contradictions found, large PDF, etc.>
   ```
6. Commit:
   ```
   git add -A
   git commit -m "ingest: <title>"
   ```

After committing, ask the user if there's a next source to process, or if the ingest pass is complete.
