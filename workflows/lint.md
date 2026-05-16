# workflows/lint.md

Health-check workflow. Read this in full before running a lint pass. Universals live in `AGENTS.md`.

## Entry point

User says: `lint`, `health check`, `audit the wiki`, `find issues`. Recommended cadence: roughly weekly, or after every ~10 ingests.

A lint pass produces a markdown report of issues found and (where safe) auto-fixes the obvious ones. It does not silently mutate curated content — anything ambiguous gets surfaced for the user to decide.

## Check list

Run each check below. For each issue found, record: what it is, which pages it affects, suggested action.

### Heuristic rules to apply alongside the checks below

Several of the checks below have a **heuristic rule that the script cannot express**. You MUST apply these rules during the corresponding check — do not rely on the script output alone. Read each item, confirm you understand it, and apply it when you reach the relevant check.

**Check 2 — classify source pages as "integration incomplete," not "orphan":**
When the orphan-detection script outputs pages under `wiki/sources/`, do NOT flag them as orphaned. Source summary pages legitimately have few inbound links until their content is cross-referenced into entities, concepts, and analyses. Instead, flag each as "integration incomplete — content not yet woven into the wiki." Only pages in `entities/`, `concepts/`, `analyses/`, and `topics/` with zero inbound links are genuine orphans.

**Check 3 — classify `[[concepts/X]]` links to nonexistent pages as deferred-concept violations:**
When the broken-wikilink script outputs `BROKEN: [[concepts/X]]` for a concept page that does not exist under `wiki/concepts/`, do NOT recommend creating the page. Per the synthesis heuristic in `workflows/ingest.md`, concepts that haven't met the 2-source promotion threshold should appear as **plain text** in source pages, not as wikilinks. A `[[concepts/X]]` link to a missing page means the ingest agent incorrectly promoted a deferred concept to a wikilink. Flag it as "deferred-concept link leak" and recommend converting to plain text.

**Check 7 — surface topic candidates manually:**
Beyond concept-level promotion, scan for groups of 3+ existing concept pages that share tags and sources — those are candidates for topic-overview pages (`wiki/topics/<theme>.md`). The promotion script catches concept-level candidates only and cannot detect topic-level clusters. When reviewing the script's PROMOTE output alongside the existing concept pages in `wiki/concepts/`, also surface any cluster of 3+ concepts sharing 2+ tags and 2+ sources as a topic candidate, with a proposed theme name for the user to confirm.

**Check 9 — cross-reference flagged log dates into source and entity pages:**
When the chronological-order script flags a date anomaly in `log.md` (out-of-order entry or wrong-year entry), do NOT stop at the log. Open the `wiki/sources/<title>.md` page named in that log entry and inspect its `created:`, `ingested:`, and `source_published:` fields for the same typo. Then check all entity and concept pages whose `created:` or `updated:` date matches the wrong date — they were likely created in the same ingest session and carry the same error. A single wrong-year log entry usually indicates a cluster of bad dates across multiple pages.

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

**CLI fallback** — scans every source page's `## Entities & concepts` section for plain-text concept names (bullets without `[[]]`), counts occurrences across sources, and flags any that appear in 2+ sources but have no `wiki/concepts/<name>.md` page. Surface candidates in the report — do not auto-promote.

```bash
python3 - <<'PY'
import os, re, glob

concept_refs = {}

for path in glob.glob("wiki/sources/*.md"):
    with open(path, encoding="utf-8") as f:
        text = f.read()
    m = re.search(r'(?s)## Entities & concepts\n(.*?)(?=\n## |\Z)', text)
    if not m:
        continue
    section = m.group(1)

    for line in section.splitlines():
        stripped = line.strip()
        if not stripped.startswith('-'):
            continue
        content = stripped.lstrip('-').strip()
        if not content or content.startswith('[['):
            continue
        # Strip bold/italic markers
        clean = re.sub(r'\*+', '', content).strip()
        # Concept name is the part before " — " (em-dash separator) or end of line
        name = clean.split(' — ')[0].split(' -- ')[0].strip()
        if name and len(name) > 2 and not os.path.exists(f"wiki/entities/{name}.md"):
            concept_refs.setdefault(name, set()).add(os.path.basename(path))

print("=== Promotion candidates (plain-text mentions in 2+ sources) ===")
promoted = False
for name in sorted(concept_refs):
    refs = concept_refs[name]
    if len(refs) >= 2 and not os.path.exists(f"wiki/concepts/{name}.md"):
        promoted = True
        print(f"PROMOTE: \"{name}\" — mentioned in {len(refs)} sources: {sorted(refs)}")
if not promoted:
    print("(none)")

print("\n=== Deferred concepts (1 source — below threshold) ===")
deferred = False
for name in sorted(concept_refs):
    refs = concept_refs[name]
    if len(refs) == 1 and not os.path.exists(f"wiki/concepts/{name}.md"):
        deferred = True
        print(f"  DEFERRED: \"{name}\" — 1 source")
if not deferred:
    print("(none)")
PY
```

Topic-cluster detection (groups of 3+ concepts sharing tags and sources) is a manual step — see the heuristic checklist at the top of this file.

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
    "source":   ["## Summary", "## Key claims", "## Entities & concepts",
                 "### Entities", "### Concepts", "## Raw"],
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

### 12b. Alias semantic scan (entity_kind: product pages)

On every `entity_kind: product` page, check whether any alias names what appears to be a distinct model or product rather than a surface form of the same entity. Per `AGENTS.md`, aliases must be different surface forms of the **same canonical thing** (e.g. "V. Bush" for "Vannevar Bush"); GPT-4 is NOT an alias for ChatGPT because they are separate entities (a model vs. a product). The script below casts a wide net — false positives are expected. Output is `REVIEW`, not `VIOLATION` — the operator decides whether each flagged alias is a legitimate surface form or should be split into its own entity page.

```bash
python3 - <<'PY'
import os, re, glob

for path in glob.glob("wiki/entities/*.md"):
    with open(path, encoding="utf-8") as f:
        text = f.read()
    m = re.match(r'(?s)^---\n(.*?)\n---', text)
    if not m:
        continue
    fm = m.group(1)
    kind = re.search(r'(?m)^entity_kind:\s*(\S+)', fm)
    if not kind or kind.group(1).strip().strip("'\"") != "product":
        continue

    base = os.path.splitext(os.path.basename(path))[0]
    aliases = []
    block = re.search(r'(?ms)^aliases:\s*\n((?:[ \t]+-\s*\S.*\n?)+)', fm)
    if block:
        for line in block.group(1).splitlines():
            a = line.strip().lstrip('-').strip().strip('"').strip("'")
            if a: aliases.append(a)
    inline = re.search(r'(?m)^aliases:\s*\[(.+)\]', fm)
    if inline:
        for a in inline.group(1).split(','):
            a = a.strip().strip('"').strip("'")
            if a: aliases.append(a)

    if not aliases:
        continue

    for alias in aliases:
        flag = False
        reasons = []
        # Pattern: contains digits (version numbers like GPT-4, GPT-5.1)
        if re.search(r'\d', alias):
            flag = True
            reasons.append("contains digits (possible version number)")
        # Pattern: hyphenated with qualifier (e.g. bart-base, bart-large, falcon-7b)
        if re.search(r'-[a-z]+$', alias, re.IGNORECASE) and not alias.lower().startswith(base.lower()):
            flag = True
            reasons.append("hyphenated qualifier suffix")
        # Pattern: slash-separated namespace (e.g. facebook/bart-base)
        if '/' in alias:
            flag = True
            reasons.append("slash-separated (namespace/variant)")
        # Pattern: common product tier keywords
        if re.search(r'\b(Pro|Sonnet|Mini|Nano|Lite|Turbo|Ultra|Max|Enterprise)\b', alias, re.IGNORECASE):
            flag = True
            reasons.append("contains product-tier keyword")
        # Pattern: entity name + distinct suffix (e.g. "ChatGPT Plus" vs "ChatGPT")
        # Only flag when the alias is NOT the page name itself and extends it
        if alias.lower() != base.lower() and alias.lower().startswith(base.lower()):
            remainder = alias[len(base):].strip().lstrip('-').strip()
            if remainder and not re.match(r'^(base|large|small|xl|xxl)$', remainder.lower()):
                # base/large/small/xl are common model size qualifiers — don't double-flag
                # if already caught by digit or hyphen check
                if not flag:
                    flag = True
                    reasons.append(f'extends entity name with "{remainder}"')

        if flag:
            print(f'REVIEW: {path} — alias "{alias}" ({", ".join(reasons)})')
            print(f'  Does "{alias}" name a distinct entity or a surface form of {base}?')
PY
```

### 13. Suspiciously empty page-type folders

A wiki with multiple ingested sources but **zero entity pages** is almost always a misapplied promotion threshold — the agent has read the 2-source rule for concepts and incorrectly applied it to entities too. Per `workflows/ingest.md`, entities have no source-count threshold; they're created on first substantive mention. The same smell applies to other page-type folders that should be non-empty given the source count.

```bash
python3 - <<'PY'
import os, glob

sources = len(glob.glob("wiki/sources/*.md"))
entities = len(glob.glob("wiki/entities/*.md"))
concepts = len(glob.glob("wiki/concepts/*.md"))

if sources >= 2 and entities == 0:
    print(f"SUSPICIOUS: {sources} source pages but 0 entity pages.")
    print("  Likely cause: agent applied the 2-source concept threshold to entities.")
    print("  Per workflows/ingest.md, entities have NO source-count threshold —")
    print("  they're created on first substantive mention.")
    print("  Review each source's `## Entities & concepts` section for deferred")
    print("  entities (people, orgs, products, places) that should have pages.")

# Soft signal: many sources but few concepts can indicate under-synthesis,
# though this is less reliable than the entities check (concepts genuinely
# need 2+ sources to promote).
if sources >= 5 and concepts == 0:
    print(f"SUSPICIOUS: {sources} source pages but 0 concept pages.")
    print("  Either the sources have no thematic overlap, or the agent has not")
    print("  yet revisited earlier sources to promote concepts that now have 2+ sources.")
PY
```

For each flagged smell: open the source pages and inspect their `## Entities & concepts` sections. Where deferred mentions describe real entities with multiple captureable facts, recommend creating entity pages. **Not a safe auto-fix** — surface the candidates with their source-context for the user to confirm before creating pages.

Also check source pages individually for empty `### Entities` blocks that contradict the source content:

```bash
python3 - <<'PY'
import re, glob

for path in glob.glob("wiki/sources/*.md"):
    with open(path, encoding="utf-8") as f:
        text = f.read()

    # Is ### Entities empty or *(none)*?
    entities_m = re.search(r'(?s)### Entities\n(.*?)(?=\n### |\n## |\Z)', text)
    if not entities_m:
        continue
    entities_body = entities_m.group(1).strip()
    if entities_body and entities_body != "*(none)*":
        continue  # has real entries — skip

    # Are there capitalized multi-word phrases in ## Key claims?
    claims_m = re.search(r'(?s)## Key claims\n(.*?)(?=\n## |\Z)', text)
    if not claims_m:
        continue
    claims = claims_m.group(1)
    proper_nouns = re.findall(r'\b[A-Z][a-z]+(?:\s+[A-Z][a-z]+)+', claims)
    unique = sorted(set(proper_nouns))
    if len(unique) >= 2:
        print(f"SUSPICIOUS EMPTY ENTITIES: {path}")
        print(f"  Key claims mention {len(unique)} capitalized phrases but ### Entities is empty.")
        print(f"  Sample: {', '.join(unique[:3])}")
PY
```

For each flagged page: open the source and inspect `## Key claims` for named persons, orgs, and products that should have entity entries. Surface candidates for user confirmation before creating pages. **Not a safe auto-fix.**

## Output

Produce a markdown report with sections per check. For each issue: severity (high / medium / low), affected pages (wikilinks), suggested action, and whether you can auto-fix.

Include `REVIEW:` items from the alias semantic scan (check 12b) as a dedicated subsection in the report. Each REVIEW item is informational unless the operator confirms a violation — do not auto-fix.

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
- checks_run: 13
- issues_found: <breakdown by severity>
- auto_fixed: <count>
- pending_user_review: <count>
```

If auto-fixes touched files, commit: `git commit -m "lint: <YYYY-MM-DD>"`. Otherwise no commit.
