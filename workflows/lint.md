# workflows/lint.md

Health-check workflow. Read this in full before running a lint pass. Universals live in `AGENTS.md`.

## Entry point

User says: `lint`, `health check`, `audit the wiki`, `find issues`. Recommended cadence: roughly weekly, or after every ~10 ingests.

A lint pass produces a markdown report of issues found and (where safe) auto-fixes the obvious ones. It does not silently mutate curated content — anything ambiguous gets surfaced for the user to decide.

## Check list

Run each check below. For each issue found, record: what it is, which pages it affects, suggested action.

### 1. `wiki/_inbox/` is non-empty

The inbox should be drained on every ingest. If files remain, list them and ask whether to ingest, delete, or leave.

```bash
ls wiki/_inbox/ 2>/dev/null
```

### 2. Orphan pages (no inbound wikilinks)

Pages nothing links to are usually dead weight or a missing-cross-reference signal.

Dataview query for `wiki/index.md` or a scratch page:
````markdown
```dataview
TABLE file.inlinks AS "Linked from"
FROM "wiki" WHERE length(file.inlinks) = 0
```
````

Source-summary pages legitimately have few inbound links until their content is integrated — flag those as "integration incomplete" instead of "orphan."

### 3. Missing entity / concept pages

Entities or concepts that appear in prose but lack their own page. Search source summaries and other pages for capitalized terms or `[[broken-wikilinks]]`:

```bash
grep -rhoE '\[\[[^]]+\]\]' wiki/ | sort -u | while read link; do
  target="${link#[[}"; target="${target%]]}"
  # check if target resolves to an existing file under wiki/
done
```

For each unresolved wikilink: suggest creating the page (entity / concept) or correcting the link.

### 4. Stale claims

Sources with `source_published` older than ~18 months on fast-moving topics. Dataview:

````markdown
```dataview
TABLE source_published, length(file.inlinks) AS "References"
FROM "wiki/sources"
WHERE source_published < date(today) - dur(18 months)
SORT length(file.inlinks) DESC
```
````

High-reference + old = highest staleness risk. Suggest a re-ingest of an updated source or a follow-up search.

### 5. Contradictions to audit

Find all `> [!conflict]` callouts in the wiki:

```bash
grep -rln '\[!conflict\]' wiki/
```

For each: confirm whether the contradiction has been reconciled or remains genuinely open. Old unreconciled conflicts are worth surfacing for the user.

### 6. Tag drift

Tags used in pages but missing from `wiki/_tags.md`:

```bash
# Extract all tags from wiki, compare with _tags.md registry
grep -rhoE '#[a-zA-Z0-9_/-]+' wiki/ | sort -u > /tmp/tags-used.txt
grep -oE '#[a-zA-Z0-9_/-]+' wiki/_tags.md | sort -u > /tmp/tags-registered.txt
comm -23 /tmp/tags-used.txt /tmp/tags-registered.txt
```

For each unregistered tag: add to `wiki/_tags.md` with a placeholder gloss for the user to refine, OR rename to an existing registered tag if a fit exists. This is a **safe auto-fix** for the registration step (the rename step is not — ask the user).

### 7. Promotion candidates

Concepts referenced by 2+ sources but lacking their own page:

````markdown
```dataview
TABLE length(file.outlinks) AS "Source refs"
FROM "wiki/sources"
FLATTEN file.outlinks AS link
GROUP BY link
WHERE length(rows) >= 2 AND !contains(link.file.path, "wiki/concepts")
```
````

Topics: clusters of 3+ related concepts without an overview page. Harder to detect automatically — surface concept pages with shared tags and many shared sources as candidates.

### 8. Image references that no longer resolve

Markdown image references pointing to files that don't exist in `raw/assets/`:

```bash
grep -rhoE '!\[.*\]\([^)]+\)' wiki/ raw/ | grep -oE '\([^)]+\)' | tr -d '()' | while read p; do
  [ -e "$p" ] || echo "broken: $p"
done
```

### 9. `log.md` chronological order and date sanity

Log entries are append-only with monotonic non-decreasing dates (newest at bottom). An out-of-order entry is usually either a stray prepend or a year typo — both worth surfacing. Date typos also propagate: when an agent writes `created: 2025-05-14` instead of `2026-05-14` in source frontmatter during an ingest, the same wrong year often ends up on every entity created in that session, so a single flagged log entry usually points to a cluster of bad dates to fix.

```bash
# Non-monotonic dates — flags any entry whose date precedes the entry before it
grep -nE '^## \[' log.md | awk -F'[][]' '
  { if (prev && $2 < prev) print "OUT OF ORDER: " $0 "  (previous: " prev ")"; prev=$2 }
'

# Year doesn't match current or previous year — usually a typo
current_year=$(date +%Y)
grep -nE '^## \[' log.md | grep -vE "\[($current_year|$((current_year-1)))-"
```

For each flagged entry: confirm the intended date with the user, then check the corresponding `wiki/sources/<title>.md` page and any entities created during that ingest for the same typo (compare `created:` / `updated:` / `ingested:` fields). This is **not** a safe auto-fix — ask before touching any date.

## Output

Produce a markdown report with sections per check. For each issue: severity (high / medium / low), affected pages (wikilinks), suggested action, and whether you can auto-fix.

Format:

```markdown
# Lint report — YYYY-MM-DD

## Summary
- High-severity: N issues
- Medium-severity: N issues
- Low-severity / informational: N issues

## 1. _inbox/ non-empty
...
```

Save the report inline in the chat. If the user wants it persisted, file it to `wiki/analyses/lint-YYYY-MM-DD.md`.

## Auto-fixes (safe set)

Apply these without asking:
- Register new tags in `wiki/_tags.md` (with a placeholder gloss noting the user should review).
- Update `wiki/index.md` with any missing one-liners for existing pages.

Ask before:
- Renaming or merging tags.
- Creating new entity / concept pages from unresolved wikilinks.
- Deleting orphan pages.
- Modifying any existing page content.

## End

Append to `log.md`:

```
## [YYYY-MM-DD] lint | <one-line summary>
- checks_run: 9
- issues_found: <breakdown by severity>
- auto_fixed: <count>
- pending_user_review: <count>
```

If auto-fixes touched files, commit: `git commit -m "lint: <YYYY-MM-DD>"`. Otherwise no commit.
