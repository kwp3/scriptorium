# workflows/ingest.md

Detailed ingest workflow. Read this in full before processing any source. Universals (page schemas, taxonomy, conventions) live in `AGENTS.md` — don't redefine them here, reference them.

## Entry point

The user invokes ingest by saying any of: `ingest`, `ingest new sources`, `process`, `add this source`, `ingest <filename>`. Files appearing in `raw/` or `wiki/_inbox/` are an implicit trigger.

### Pending-source detection (when invoked without a filename)

1. List everything in `raw/` (root + subfolders, excluding `assets/`) and `wiki/_inbox/`.
2. Diff against `log.md`: anything without a matching `ingest: <title>` entry is pending.
3. Cross-check by looking for missing `wiki/sources/<title>.md` pages (defense-in-depth — files can be renamed, log entries can be wrong).
4. Present the pending list to the user. Confirm before starting.
5. Process one source at a time. One ingest = one commit.

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

### b) PDFs (seamless workflow — the priority pain point)

1. PDF lands in `raw/pdfs/<title>.pdf` (route from `raw/` root if it landed loose).
2. Extract text and tables using the **highest-quality available** PDF tool. Priority order:
   - `anthropic-skills:pdf` skill (best, if running Claude Code with skills available)
   - `marker` (https://github.com/datalab-to/marker) — best open-source PDF→markdown for academic papers
   - `docling` (https://github.com/docling-project/docling) — IBM, strong layout preservation
   - `pymupdf` (`fitz`) — fast, programmatic, no OCR
   - `pdftotext` (poppler) — last resort, plain text only
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
  - A concept gets its own `wiki/concepts/` page once **2+ sources** reference it.
  - A topic / theme overview gets created once **3+ concepts** cluster around it.
- **No new info without a source.** Never fabricate connections or claims. Every assertion in `wiki/` traces back to a file under `raw/`.
- **Prefer updates over creation.** If an entity / concept page already exists, update it. Only create when no match exists.
- **Deferred concepts are PLAIN TEXT, not wikilinks.** A concept that hasn't met the 2-source promotion threshold gets mentioned as plain text in the source's `## Entities & concepts` section and anywhere else in prose — e.g. `... see Diffusion Model Prompting for more (1 source so far, page deferred)`. Do NOT write `[[concepts/Diffusion Model Prompting]]` while the page doesn't exist; Obsidian will show it as a broken link and lint will flag it. Promote to wikilink only when you create the page.
- **Cross-link new and updated concept pages.** When you create or update a `wiki/concepts/<name>.md` page, add `[[concepts/<name>]]` to the `## Related` section of every entity referenced on that concept page. Backlinks are implicit in Obsidian's graph, but a visible Related section helps readers and lint.

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
