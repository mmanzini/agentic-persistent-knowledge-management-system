# Vault schema — the frozen harness

This file is the **contract** by which the wiki is structured and
judged. It is the autoresearch `prepare.py` analog: read-only to the
agent, edited only by the human.

**The agent MUST NOT modify this file during `consolidate`, `refine`,
or `evaluate`.** Changes to this file are a deliberate human decision
— the equivalent of changing the evaluation harness in autoresearch.
If a verb run requires a schema change to succeed, that's a signal
to stop the run and surface the question to the human, not to edit
the schema.

The agent reads this file once at session start and treats it as
fixed for the rest of the session.

---

## Hard rules

These are non-negotiable. A run that violates any of them is a bug.

1. **`Resources/` is immutable** during read. The only writes are
   post-`consolidate` deletions of sources whose parent folder has
   `delete_after_consolidation: true` (or unset).
2. **`[[wiki links]]` are same-bucket only.** Free across topics
   inside a bucket; **never** across buckets. An article that needs
   to exist in two buckets is duplicated, not linked.
3. **No silent bucket creation.** Sources matching no existing bucket
   go to `Intelligence/_unsorted/` and surface in the run report.
4. **Topic creation is allowed and expected.** When no existing topic
   in the chosen bucket fits, create one with its `_index.md` and
   register it in the bucket's `_master-index.md`.
5. **Images live with their articles.** Every `![[...]]` embed must
   resolve to a file in the same topic folder. No cross-folder image
   references. Duplicating an article into two buckets means copying
   the image into each.
6. **Indexes and log stay in sync, every run.** A `consolidate` that
   touches a bucket must update the topic's `_index.md` (always), the
   bucket's `_master-index.md` (if a topic was created), and
   `Intelligence/log.tsv` (always — one append per processed source).
   `Intelligence/index.md` only changes on first-touch of a bucket.
   The `query` verb relies on these indexes being truthful — stale
   indexes silently degrade retrieval, so this is enforced as a hard
   rule, not a courtesy. `refine` flags any drift it finds.
7. **File names are `lowercase-with-hyphens`.** Both folders and
   files. Article slugs match this convention.
8. **Single locus of change.** During a parallel `consolidate` or
   `refine`, each per-bucket subagent writes **only** inside its
   assigned `Intelligence/<bucket>/` subtree. The **orchestrator** is
   the sole writer of `Intelligence/index.md`,
   `Intelligence/log.tsv`, `Intelligence/_unsorted/`, and
   `Intelligence/_eval/results.tsv`. This eliminates write races and
   matches autoresearch's "one file is the locus of change" principle
   — adapted to a per-bucket fan-out.
9. **Schema is frozen.** This file (`Vault/schema.md`) is read-only
   to the agent. No verb may edit it.

---

## Schemas

Don't paraphrase these. Copy them verbatim when generating files.

### `Resources/<folder>/README.md`

```markdown
---
include_in_consolidation: true     # false → folder is skipped during consolidate
delete_after_consolidation: true   # false → source files stay in place even after successful consolidation
---

# <Folder name>

One paragraph describing what lives here (transcriptions, web
clippings, ideas, etc.) and any handling notes (e.g. "treat each .md
as a single source").
```

The two flags are independent:

- `include_in_consolidation: false` → the folder is dormant input,
  kept for human reference only.
- `delete_after_consolidation: false` → sources are read and
  consolidated, but the originals are **kept** in `Resources/` after
  a successful run. Use this for material with archival value of its
  own (meeting transcriptions, primary documents).
- `delete_after_consolidation: true` → sources are read,
  consolidated, and **deleted** from `Resources/` once `consolidate`
  steps 1–8 succeed. Use for ephemeral material like web clippings.
- Default if either field is missing: `include_in_consolidation: true`,
  `delete_after_consolidation: true`.

### `Intelligence/<bucket>/_master-index.md`

```markdown
# <Bucket name>

**Scope**: One paragraph describing what belongs in this bucket and
what does not. This is the routing signal the agent uses when
allocating new articles.

## Topics

- [[<topic>/_index|Topic Title]] — one-line description of the topic cluster
- ...
```

### `Intelligence/<bucket>/<topic>/_index.md`

```markdown
# <Topic name>

One paragraph describing what this topic clusters.

## Articles

- [[<article-slug>|Article Title]] — one-line description
- ...

## Related Topics

- [[../<other-topic>/_index|Other Topic]] — one-line description
- ...
```

`Related Topics` may link to other topics **within the same bucket only**.

### Article schema

```markdown
# <Article title>

**Source:** [<source name>](<url-or-path>)
**Author:** <if known>
**<Other top-of-page metadata as relevant>**

---

## Summary

One short paragraph — what this article is about and the core thesis.

## <Body section>

Body content. Cite each factual claim inline with `(source: <path-or-filename>)`.

Embed images with `![[image.png]]`. The image file must live in the
same topic folder as the article — see *Image handling* below.

## Key Takeaways

- ...

## Related

- [[<other-article-in-this-bucket>]] — one-line note
```

---

## Citation rules

These apply to every article body. They are part of the schema, not
a suggestion.

- **Every factual claim references its source file.** No bare claims.
- **Format: `(source: filename.pdf)`** (or `.md`, `.html`, etc.)
  **after the claim**, inline. Not as a footnote, not at section end.
- **If two sources disagree, note the contradiction explicitly** in
  the article body. Example: *"Source A says X (source: a.md);
  source B says Y (source: b.md) — unresolved."* `consolidate`
  surfaces contradictions inline; `refine` reports them.
- **If a claim has no source, mark it `(source: needs-verification)`**
  so `refine` can pick it up. Never invent a source path to silence
  this rule.

---

## Image handling

When a source file references an image (or the article body needs
one):

1. Locate the image in `Resources/` (it should sit beside the source
   file that references it).
2. **Copy** the image file into the article's topic folder
   (`Intelligence/<bucket>/<topic>/`). Don't link out to `Resources/`
   — articles must remain self-contained so deleting the source after
   consolidation doesn't break embeds.
3. Reference the image with `![[image-filename.ext]]` — Obsidian
   resolves it from the same folder.
4. If the same image is referenced by articles in two different
   topics (or two buckets) during a duplication, copy it into each
   topic folder. Storage cost is trivial; the self-containment
   property matters more.

When a source is deleted post-consolidation, its referenced images
are deleted alongside it. When a source is kept, its images stay too
— the topic folder already has its own copy from step 4 of
`consolidate`.

---

## Log format — `Intelligence/log.tsv`

Tab-separated, fixed columns. Header row is line 1. One append per
processed source for `consolidate`; one summary row per run for
`refine` and `evaluate`.

Columns (9, in order):

```
timestamp	verb	source	bucket	topic	articles_changed	images_copied	status	notes
```

- `timestamp` — ISO 8601 UTC (`2026-04-26T14:32:01Z`).
- `verb` — `consolidate` | `refine` | `evaluate`.
- `source` — path under `Resources/` for `consolidate` rows;
  `(refine)` for refine summary rows; `(evaluate)` for evaluate
  summary rows.
- `bucket` — the bucket touched, `_unsorted` for quarantine,
  `(all)` for cross-bucket summary rows, empty if not applicable.
- `topic` — the topic touched, empty if not applicable.
- `articles_changed` — integer count (written + updated).
- `images_copied` — integer count.
- `status` — `kept` | `quarantined` | `crashed` | `refine_summary`
  | `eval_summary`.
- `notes` — short freeform. For `refine_summary` rows, **must
  include** `drift=<N>` where N is the total drift count
  (orphans + index-mismatches + broken-embeds + cross-bucket-links
  + schema-violations). For `eval_summary` rows, **must include**
  `total_files_read=<N>` and `quality=<good>/<partial>/<poor>/<missing>`
  counts (e.g. `quality=3g/1p/1m`).

**Never use commas as separators.** Tabs only — commas appear in
freeform `notes`.

---

## Eval results format — `Intelligence/_eval/results.tsv`

Tab-separated, fixed columns. Header row is line 1. One append per
question per `evaluate` run.

Columns (5, in order):

```
timestamp	question_id	files_read	answer_quality	notes
```

- `timestamp` — ISO 8601 UTC.
- `question_id` — `q1`, `q2`, …, matching IDs in
  `_eval/questions.md`.
- `files_read` — integer count of files the agent had to `Read` to
  answer (excludes `index.md`, `_master-index.md`, and `_index.md`
  reads — those are routing overhead, not retrieval cost). The
  primary metric — the closest analog to autoresearch's `val_bpb`.
  Lower is better.
- `answer_quality` — `good` | `partial` | `poor` | `missing`.
  Self-judged by the agent against the question's intent.
- `notes` — short freeform. Note any cross-topic walks, any reliance
  on `(source: needs-verification)` claims, any quarantine hits.

---

## Drift count — what `refine` measures

A single integer. Sum of:

- **Orphans** — articles with zero inbound `[[wiki links]]` from
  other articles in their bucket (excluding the topic's own
  `_index.md` listing).
- **Index mismatches** — entries in any `_index.md`,
  `_master-index.md`, or `index.md` that don't resolve to a real
  file, or files on disk not listed in the relevant index.
- **Broken embeds** — `![[image.ext]]` references that don't resolve
  in the same topic folder.
- **Cross-bucket links** — any `[[link]]` that crosses bucket
  boundaries (hard-rule violation; counts as a single drift unit per
  occurrence).
- **Schema violations** — articles missing required sections
  (`Source:`, `Summary`, etc.), or factual claims without
  `(source: ...)` citations.

**Advance-on-improvement rule:** the orchestrator compares the
current `refine_summary` row's drift count to the previous one (last
`refine_summary` row in `log.tsv`). It reports the delta as
**advance** (drift down, or drift flat with `articles_changed > 0`),
**hold** (drift flat with `articles_changed == 0`), or **regress**
(drift up). The orchestrator never silently undoes work — humans
decide what to do with a regression report.
