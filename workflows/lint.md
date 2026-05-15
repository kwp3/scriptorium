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

**Obsidian / Dataview** (manual review inside Obsidian):
````markdown
```dataview
TABLE file.inlinks AS "Linked from"
FROM "wiki" WHERE length(file.inlinks) = 0
```
````

**CLI fallback** (for agents running from a terminal — Dataview only works inside Obsidian):
```bash
# List every page; for each, count how many other pages wikilink to its basename
for f in wiki/sources/*.md wiki/entities/*.md wiki/concepts/*.md wiki/analyses/*.md wiki/topics/*.md; do
  [ -f "$f" ] || continue
  base=$(basename "$f" .md)
  # Match either [[base]] or [[<subdir>/base]]; exclude self-matches
  count=$(grep -rlF -e "[[$base]]" -e "[[$base|" -e "/$base]]" -e "/$base|" wiki/ 2>/dev/null | grep -vF "$f" | wc -l)
  [ "$count" -eq 0 ] && echo "ORPHAN: $f"
done
```

Source-summary pages legitimately have few inbound links until their content is integrated — flag those as "integration incomplete" instead of "orphan."

### 3. Missing entity / concept pages (broken wikilinks)

Entities or concepts referenced as `[[Wikilink]]` but whose page doesn't exist. The resolver below handles all three wikilink shapes the spec uses:

- **Prefixed**, e.g. `[[concepts/X]]` → must resolve to `wiki/concepts/X.md`.
- **Bare**, e.g. `[[Gemini]]` → must resolve to `wiki/<any subdir>/Gemini.md` (entities, concepts, sources, analyses, topics).
- **Aliased display**, e.g. `[[Wikipedia|WikiProject AI Cleanup]]` → resolve the part before the pipe.

```bash
grep -rhoE '\[\[[^]]+\]\]' wiki/ | sort -u | while read link; do
  inner="${link#[[}"; inner="${inner%]]}"
  target="${inner%%|*}"                       # strip "|display-text" if present
  if echo "$target" | grep -qE '^(concepts|entities|sources|analyses|topics)/'; then
    [ -f "wiki/$target.md" ] || echo "BROKEN: $link"
  else
    found=0
    for sub in entities concepts sources analyses topics; do
      [ -f "wiki/$sub/$target.md" ] && found=1 && break
    done
    [ "$found" -eq 0 ] && echo "BROKEN: $link"
  fi
done
```

Resolution rules: filenames are case-sensitive on Linux/macOS, case-insensitive on Windows — assume case-sensitive. A `[[concepts/Deferred Concept]]` link to a page that doesn't exist is a synthesis-heuristic violation (deferred concepts should be plain text, not wikilinks — see `workflows/ingest.md`); flag it as such rather than recommending the page be created.

For each unresolved wikilink: suggest creating the page (if it should exist) or correcting the link (if a deferred-concept reference leaked through as a wikilink).

### 4. Stale claims

Sources with `source_published` older than ~18 months on fast-moving topics.

**Obsidian / Dataview** (manual review):

````markdown
```dataview
TABLE source_published, length(file.inlinks) AS "References"
FROM "wiki/sources"
WHERE source_published < date(today) - dur(18 months)
SORT length(file.inlinks) DESC
```
````

**CLI fallback** (Python — bash date arithmetic is awkward):
```bash
python3 - <<'PY'
import os, re, datetime as dt, glob
cutoff = dt.date.today() - dt.timedelta(days=18*30)
for path in glob.glob("wiki/sources/*.md"):
    with open(path, encoding="utf-8") as f:
        text = f.read()
    m = re.search(r'(?m)^source_published:\s*(\S+)', text)
    if not m: continue
    val = m.group(1).strip().strip('"').strip("'")
    if val in ("unknown", "null"): continue
    try:
        d = dt.date.fromisoformat(val[:10])
    except ValueError:
        continue
    if d < cutoff:
        print(f"STALE: {path} (published {val})")
PY
```

High-reference + old = highest staleness risk. Suggest a re-ingest of an updated source or a follow-up search.

### 5. Contradictions to audit

Find all `> [!conflict]` callouts in the wiki:

```bash
grep -rln '\[!conflict\]' wiki/
```

For each: confirm whether the contradiction has been reconciled or remains genuinely open. Old unreconciled conflicts are worth surfacing for the user.

### 6. Tag drift

Tags used in page frontmatter but missing from `wiki/_tags.md`.

**Why the naive `grep '#tag'` approach is wrong:** tags live in YAML frontmatter as **bare strings** under the `tags:` key, not as `#hashtags` in body text. And `wiki/_tags.md` registers them **backtick-wrapped** (`` `article` ``), not `#`-prefixed. A `grep '#[a-zA-Z]'` against either side will give garbage (false positives from body-text hashtags, false negatives on the registry). The check below uses a YAML-aware Python parser instead.

```bash
python3 - <<'PY'
import os, re, glob

# 1. Collect tags from every page's YAML frontmatter
used = set()
for path in glob.glob("wiki/**/*.md", recursive=True):
    if path.endswith("_tags.md") or path.endswith("index.md"):
        continue
    with open(path, encoding="utf-8") as f:
        text = f.read()
    m = re.match(r'(?s)^---\n(.*?)\n---', text)
    if not m: continue
    fm = m.group(1)
    # tags can be a list block or inline [a, b, c]
    block = re.search(r'(?ms)^tags:\s*\n((?:[ \t]+-\s*\S.*\n?)+)', fm)
    if block:
        for line in block.group(1).splitlines():
            tag = line.strip().lstrip('-').strip().strip('"').strip("'")
            if tag: used.add(tag)
    inline = re.search(r'(?m)^tags:\s*\[(.+)\]', fm)
    if inline:
        for tag in inline.group(1).split(','):
            tag = tag.strip().strip('"').strip("'")
            if tag: used.add(tag)

# 2. Collect tags from _tags.md (strip backticks)
registered = set()
with open("wiki/_tags.md", encoding="utf-8") as f:
    for line in f:
        for m in re.finditer(r'`([^`]+)`', line):
            registered.add(m.group(1))

# 3. Diff
for tag in sorted(used - registered):
    print(f"UNREGISTERED: {tag}")
for tag in sorted(registered - used):
    print(f"UNUSED IN REGISTRY: {tag}")
PY
```

For each unregistered tag: add to `wiki/_tags.md` with a placeholder gloss for the user to refine, OR rename to an existing registered tag if a fit exists. This is a **safe auto-fix** for the registration step (the rename step is not — ask the user). Unused-in-registry entries are usually fine (pre-registered slots like `transcript` and `note` may not yet have any sources) but worth surfacing in case a tag was renamed but the old entry was left behind.

### 7. Promotion candidates

Concepts referenced by 2+ sources (as either wikilinks or plain text) but lacking their own page.

**Obsidian / Dataview** (manual review; works against wikilinks only):

````markdown
```dataview
TABLE length(file.outlinks) AS "Source refs"
FROM "wiki/sources"
FLATTEN file.outlinks AS link
GROUP BY link
WHERE length(rows) >= 2 AND !contains(link.file.path, "wiki/concepts")
```
````

**CLI fallback** — the harder case is plain-text mentions (deferred concepts), since those aren't wikilinks. The reliable heuristic is: scan every source page's `## Entities & concepts` section for plain-text concept names, count occurrences across sources, flag any that appear in 2+ sources but have no `wiki/concepts/<name>.md` page. This is best done by a human reviewer reading the report — surface candidates rather than auto-promoting.

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

### 10. Duplicate entities

Two pages describing the same canonical thing under different names — e.g. `entities/Google Gemini.md` and `entities/Gemini.md`. The signal that's easy to detect mechanically: each page lists the other's name in its `aliases:` block, or both pages share 2+ aliases.

```bash
python3 - <<'PY'
import os, re, glob

pages = {}   # basename -> set of aliases (lowercased)
for path in glob.glob("wiki/entities/*.md") + glob.glob("wiki/concepts/*.md"):
    base = os.path.splitext(os.path.basename(path))[0]
    with open(path, encoding="utf-8") as f:
        text = f.read()
    m = re.match(r'(?s)^---\n(.*?)\n---', text)
    if not m: continue
    fm = m.group(1)
    aliases = set()
    block = re.search(r'(?ms)^aliases:\s*\n((?:[ \t]+-\s*\S.*\n?)+)', fm)
    if block:
        for line in block.group(1).splitlines():
            a = line.strip().lstrip('-').strip().strip('"').strip("'")
            if a: aliases.add(a.lower())
    inline = re.search(r'(?m)^aliases:\s*\[(.+)\]', fm)
    if inline:
        for a in inline.group(1).split(','):
            a = a.strip().strip('"').strip("'")
            if a: aliases.add(a.lower())
    pages[base] = aliases

# Pairwise: A's name in B's aliases (or vice versa), or 2+ shared aliases
names = list(pages.keys())
seen = set()
for i, a in enumerate(names):
    for b in names[i+1:]:
        key = tuple(sorted([a, b]))
        if key in seen: continue
        cross = (a.lower() in pages[b]) or (b.lower() in pages[a])
        shared = pages[a] & pages[b]
        if cross or len(shared) >= 2:
            seen.add(key)
            reason = "cross-referenced aliases" if cross else f"share {len(shared)} aliases ({sorted(shared)})"
            print(f"DUP CANDIDATE: {a} <-> {b}  ({reason})")
PY
```

For each flagged pair: ask the user which name is canonical, then merge — move the loser's facts into the winner, add the loser's name to the winner's `aliases:`, delete the loser's page, and `grep -rl '[[loser]]'` to rewrite any references. **Not a safe auto-fix** — always ask.

### 11. Index completeness

Every page under `wiki/sources/`, `wiki/entities/`, `wiki/concepts/`, `wiki/analyses/`, `wiki/topics/` should appear in `wiki/index.md` exactly once.

```bash
python3 - <<'PY'
import os, re, glob

# Pages that should be indexed
pages = set()
for sub in ("sources", "entities", "concepts", "analyses", "topics"):
    for path in glob.glob(f"wiki/{sub}/*.md"):
        base = os.path.splitext(os.path.basename(path))[0]
        pages.add((sub, base))

# Wikilinks present in index.md
with open("wiki/index.md", encoding="utf-8") as f:
    idx = f.read()
indexed = set()
for m in re.finditer(r'\[\[([^]|]+)(?:\|[^]]+)?\]\]', idx):
    inner = m.group(1)
    if '/' in inner:
        sub, base = inner.split('/', 1)
        indexed.setdefault((sub, base), 0)
    else:
        # Bare wikilink — assign to whichever subdir has the file
        for sub, base in pages:
            if base == inner:
                indexed.setdefault((sub, base), 0)
                break

# Count duplicates (a page listed twice in index.md)
counts = {}
for m in re.finditer(r'\[\[([^]|]+)(?:\|[^]]+)?\]\]', idx):
    inner = m.group(1)
    target = inner.split('/')[-1]
    counts[target] = counts.get(target, 0) + 1

for sub, base in sorted(pages):
    if (sub, base) not in indexed:
        print(f"NOT IN INDEX: {sub}/{base}")
for target, c in counts.items():
    if c > 1 and any(base == target for _, base in pages):
        print(f"DUPLICATE IN INDEX: {target} (appears {c}x)")
PY
```

**Safe auto-fix**: append a one-liner entry under the correct section for any page missing from the index. For duplicates, dedupe by keeping the entry with the longer one-line summary; if both are equal, ask.

### 12. Frontmatter schema validation

Every page must have the required fields for its type per the schemas in `AGENTS.md`. The most common failure modes:

- Source pages with off-schema frontmatter (e.g. Web Clipper's `title`/`date`/`url`/`authors` left in place instead of being rewritten to `type: source`/`created`/`ingested`/`source_url`/`source_published`).
- Entity pages using `date:` instead of `created:`/`updated:`.
- `entity_kind` values outside the allowed enum (`person | org | product | place | other`).
- Missing required sections (e.g. concept page without `## Definition`).

```bash
python3 - <<'PY'
import os, re, glob

REQUIRED_FIELDS = {
    "source":   ["type", "created", "ingested", "source_type", "source_url", "source_published", "tags"],
    "entity":   ["type", "entity_kind", "aliases", "created", "updated", "sources", "tags"],
    "concept":  ["type", "created", "updated", "sources", "tags"],
    "analysis": ["type", "created", "question", "sources", "tags"],
    "topic":    ["type", "created", "updated", "entities", "concepts", "tags"],
}
REQUIRED_SECTIONS = {
    "source":   ["## Summary", "## Key claims", "## Entities & concepts", "## Raw"],
    "entity":   ["## Overview", "## Key facts", "## Related"],
    "concept":  ["## Definition", "## Origins / proponents", "## Examples", "## Critiques or contradictions"],
    "analysis": ["## Question", "## Findings", "## Caveats / gaps"],
    "topic":    ["## Overview", "## Key entities", "## Key concepts", "## Open questions"],
}
ENUMS = {
    "source_type": {"article", "pdf", "transcript", "note", "other"},
    "entity_kind": {"person", "org", "product", "place", "other"},
}
SUBDIR_TO_TYPE = {"sources": "source", "entities": "entity", "concepts": "concept",
                  "analyses": "analysis", "topics": "topic"}

for path in glob.glob("wiki/**/*.md", recursive=True):
    parts = path.replace("\\", "/").split("/")
    if len(parts) < 3: continue
    subdir = parts[1]
    if subdir not in SUBDIR_TO_TYPE: continue
    expected_type = SUBDIR_TO_TYPE[subdir]

    with open(path, encoding="utf-8") as f:
        text = f.read()
    m = re.match(r'(?s)^---\n(.*?)\n---(.*)', text)
    if not m:
        print(f"NO FRONTMATTER: {path}")
        continue
    fm, body = m.group(1), m.group(2)

    # Field presence
    for field in REQUIRED_FIELDS[expected_type]:
        if not re.search(rf'(?m)^{field}\s*:', fm):
            print(f"MISSING FIELD {field}: {path}")

    # Enum values
    for field, allowed in ENUMS.items():
        mm = re.search(rf'(?m)^{field}\s*:\s*(\S+)', fm)
        if mm:
            val = mm.group(1).strip().strip('"').strip("'")
            if val not in allowed:
                print(f"BAD ENUM {field}={val} (allowed: {sorted(allowed)}): {path}")

    # Section headers
    for section in REQUIRED_SECTIONS[expected_type]:
        if section not in body:
            print(f"MISSING SECTION '{section}': {path}")

    # Source-type tag must match source_type frontmatter
    if expected_type == "source":
        mm = re.search(r'(?m)^source_type\s*:\s*(\S+)', fm)
        if mm:
            st = mm.group(1).strip().strip('"').strip("'")
            if st in ("article", "pdf", "transcript", "note"):
                # tags block contains st?
                tags_block = re.search(r'(?ms)^tags:\s*\n((?:[ \t]+-\s*\S.*\n?)+)', fm)
                tag_set = set()
                if tags_block:
                    for line in tags_block.group(1).splitlines():
                        t = line.strip().lstrip('-').strip().strip('"').strip("'")
                        if t: tag_set.add(t)
                if st not in tag_set:
                    print(f"MISSING SOURCE-TYPE TAG '{st}' in tags: {path}")
PY
```

For each issue: surface in the report, ask the user before rewriting frontmatter (the agent may have intentionally added fields beyond the schema — those are allowed, but missing required fields and bad enum values almost always indicate ingest-time error). **Not a safe auto-fix.**

A separate, more heuristic spot-check: on every `entity_kind: product` page that represents a chatbot or model family, scan `aliases:` for entries that look like model versions (`<entity>-<digit>`, `GPT-4`, `<entity> Pro`, `<entity> Sonnet`, etc.). These are usually distinct entities, not surface forms — the spec calls this out explicitly in `AGENTS.md`. Surface for user review.

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
- checks_run: 12
- issues_found: <breakdown by severity>
- auto_fixed: <count>
- pending_user_review: <count>
```

If auto-fixes touched files, commit: `git commit -m "lint: <YYYY-MM-DD>"`. Otherwise no commit.
