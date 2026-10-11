# Judged bulk edge repair (linking isolated pages)

Use when the user wants a class of isolated pages (atoms, day notes, imported
stubs) linked into the graph and no existing pipeline creates those edges.
Order is fixed: propose → judge → apply → verify → hand over the revert.

## 0. Pick a declared link type first

```bash
gbrain link <some-page> <some-concept> --link-type <T> --link-source <batch-source>
```

Run it once on a real pair and read the error. `gbrain link` refuses types the
active pack does not declare and the error enumerates the declared ones — the
pipeline's own `synthesized_from` / `synthesizes` are internal-only and are
rejected here. `discusses` is the safe weak claim ("this page is about that
topic"); `relates_to` is weaker still; anything that asserts provenance or a
merge needs the evidence to back it.

The edge is ONE write per source page: enough to de-isolate it. Multiple edges
per page are a bonus, not a requirement — extra edges multiply the judging work.

## 1. Propose candidates in SQL (cheap, local, no LLM)

Embeddings live in `content_chunks` (the `pages` table has no content column),
so join on it. Swap the two type filters for the classes being connected.

```bash
export PGC="$(python3 -c "import json;print(json.load(open('$HOME/.gbrain/config.json'))['database_url'])")"
psql "$PGC" -t -A -F$'\t' -o "$SCRATCH/candidates.tsv" -c "
with src as (
  select p.id, p.slug, p.title, p.frontmatter->'concepts' refs, c.embedding
  from pages p join content_chunks c on c.page_id = p.id
  where p.deleted_at is null and p.type = 'atom'
    and not exists (select 1 from links l
                    where l.from_page_id = p.id or l.to_page_id = p.id)),
tgt as (
  select p.slug, c.embedding from pages p join content_chunks c on c.page_id = p.id
  where p.deleted_at is null and p.type = 'concept')
select src.slug, left(src.title, 68),
  (select string_agg(t, ',') from jsonb_array_elements_text(src.refs) t),
  x.s1, round((1 - x.d1)::numeric, 3)::text, x.s2, round((1 - x.d2)::numeric, 3)::text
from src, lateral (
  select (array_agg(slug order by d))[1] s1, (array_agg(d order by d))[1] d1,
         (array_agg(slug order by d))[2] s2, (array_agg(d order by d))[2] d2
  from (select tgt.slug, src.embedding <=> tgt.embedding d from tgt
        order by src.embedding <=> tgt.embedding limit 2) k) x
order by src.slug;"
```

Read the file with a line-numbering reader — the printed line number IS the
verdict key in the next step.

## 2. Judge, don't threshold

Score bands are a hint, not a rule. In practice the top hits are mostly right
and the tail is visibly wrong (a sandboxed-browser page landing on
`shell-path-resolution`, a reranker page landing on `context-budget`). Judge
"is this page about that concept's subject", skip when nothing shares the
subsystem, and name the skipped ones in the report — an isolated page is
cheaper than a wrong edge. Two traps:

- A high score does not beat a more specific concept. Prefer the candidate
  whose topic is narrower when both are plausible.
- A concept page can be a better target than anything in the top-2 (the right
  parent is sometimes third because its stub text is thin). You may add such a
  target by hand; that is judgment, not a threshold.

## 3. Verdicts keyed by line number

`<row-number>\t<target-slug>`, one per line, skipping the rejected rows. Long
slugs then never get retyped.

## 4. Applier (idempotent, dry-run by default)

```python
#!/usr/bin/env python3
import subprocess, sys

SCRATCH = "/path/to/scratch"
rows = [l.rstrip("\n").split("\t") for l in open(f"{SCRATCH}/candidates.tsv") if l.strip()]
verdicts = {int(i): c.strip() for i, c in
            (l.split("\t") for l in open(f"{SCRATCH}/verdicts.tsv") if l.strip())}

assert rows and max(verdicts) <= len(rows), "verdict index does not match proposal file"
dry = "--apply" not in sys.argv
ok = fail = 0
for i in sorted(verdicts):
    slug, target = rows[i - 1][0], verdicts[i]
    assert slug.startswith("atoms/") and target.startswith("concepts/"), (slug, target)
    if dry:
        print("gbrain link", slug, target, "--link-type discusses --link-source <batch-source>")
        continue
    r = subprocess.run(["gbrain", "link", slug, target, "--link-type", "discusses",
                        "--link-source", "<batch-source>"], capture_output=True, text=True)
    if r.returncode == 0 and '"ok"' in r.stdout:
        ok += 1
    else:
        fail += 1
        print("FAIL", slug, target, (r.stdout + r.stderr)[-200:], file=sys.stderr)
print(f"{'would link' if dry else 'linked'}: {len(verdicts) if dry else ok}  failed: {fail}")
```

Keep the script: re-editing one verdict line and re-running is how a wrong
judgment gets fixed.

## 5. Verify

```sql
-- did the isolated count drop?
select count(*) from pages p where p.deleted_at is null and p.type = 'atom'
  and not exists (select 1 from links l where l.from_page_id = p.id or l.to_page_id = p.id);
-- batch size, by type+source
select link_type, link_source, count(*) from links where link_source = '<batch-source>' group by 1,2;
```

```bash
gbrain graph <one-changed-slug> --depth 1   # the page now shows the edge, and the target gains the back edge
```

## 6. Revert (always state it)

```sql
delete from links where link_source = '<batch-source>';     -- bulk
```

```bash
gbrain unlink <from> <to> --link-source <batch-source>     # per edge (needs both slugs)
```

## Reporting

Give: how many linked, how many deliberately skipped and why (topics, not
counts alone), how many edges are loose (same topic, different mechanism —
that is what `discusses` means, and say so), that no page body was rewritten,
and the revert statement. Do not claim the graph metric moved: for machine-leaf
types it will not (see the orphan-policy section in SKILL.md).
