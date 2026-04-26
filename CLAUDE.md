# Vault — single Resources zone, two-layer Intelligence

This vault is one wiki-backed RAG. Two zones, two roles:

- **`Resources/`** — immutable input. Raw source material (web clippings,
  transcriptions, ideas, primary documents). The agent reads from here
  and may delete sources after consolidating them, but otherwise never
  modifies anything in this tree.
- **`Intelligence/`** — the wiki. Long-term memory the agent maintains.
  Two layers inside it:
  - **Buckets** (`Intelligence/<bucket>/`) — **user-curated** macro
    groups. Each has a `_master-index.md` declaring its scope. The
    agent **never** invents a new bucket. If nothing fits, the source
    goes to `_unsorted/`.
  - **Topics** (`Intelligence/<bucket>/<topic>/`) — **agent-curated**
    clusters created during `consolidate`. Topic creation is allowed
    and expected.

The verbs below (`query`, `consolidate`, `refine`) are the only contract.
Follow them verbatim.

---

## Verb: `query`

This is the actual point of the wiki — let an agent pull only the
relevant context without dragging the whole tree into its window.
**Walk the indexes top-down, read article bodies last.**

1. **Start at `Intelligence/index.md`.** Read it cold to learn which
   buckets exist and what each one covers.
2. **Pick the bucket(s).** Match the question against the one-line
   bucket descriptions and, if needed, the bucket's `_master-index.md`
   Scope paragraph. If no bucket fits, say so and stop. Don't guess
   into `_unsorted/` — it's a quarantine, not a search target unless
   the user explicitly asks.
3. **Pick the topic(s) within the chosen bucket(s).** Read the
   bucket's `_master-index.md` Topics list and, if disambiguation is
   needed, the candidate topics' `_index.md` description paragraphs.
   **Do not load article bodies yet.**
4. **Read articles.** Within the chosen topic(s), use the topic
   `_index.md` Articles list (each article has a one-line description)
   to pick which article files to actually `Read`. Pull only those.
   The topic's `Related Topics` block tells you where to step sideways
   within the same bucket if the answer leaks across topics.
5. **Stop at bucket boundaries.** Wiki links inside articles are
   guaranteed same-bucket only, so following `[[wiki links]]` during
   query is safe. If a question genuinely spans buckets, repeat the
   process per bucket and return parallel answers — never assume
   cross-bucket continuity.
6. **Cite back.** Every fact in the answer cites the article path it
   came from, in the same `(source: <path>)` form articles use
   internally.
7. **Budget rule.** Default ceiling: read at most one `index.md` +
   one or two `_master-index.md` + a handful of `_index.md` + the
   article bodies actually needed. If a question seems to require
   reading more than 5 article bodies, surface that to the user before
   continuing — usually the question is too broad or the buckets are
   mis-cut.

---

## Verb: `consolidate`

When the user says "consolidate":

1. Walk `Resources/` recursively. For each subfolder, read its
   `README.md` and skip if `include_in_consolidation: false`.
2. For every remaining source file, pick the best-fit **bucket(s)** by
   comparing content against each `Intelligence/<bucket>/_master-index.md`
   Scope paragraph.
   - If no existing bucket fits, write the article into
     `Intelligence/_unsorted/` and flag it in the run report. **Never
     silently invent a top-level bucket** — bucket creation is a user
     decision, surfaced via the quarantine.
   - If two or more buckets fit, write the article into each (with
     images copied into each — see *Image handling* below).
3. Within the chosen bucket, pick the best-fit **topic** by comparing
   content against each `<topic>/_index.md` description. If no
   existing topic fits, **create a new topic folder** with its
   `_index.md` and add it to the bucket's `_master-index.md`. Topic
   creation is the agent's job; bucket creation is not.
4. Write or update the article using the *Article schema* below. Copy
   any referenced images into the topic folder per *Image handling*.
5. Update the topic's `_index.md` (add the new article, refresh
   `Related Topics` if needed).
6. Update the bucket's `_master-index.md` if a new topic was created.
7. Update `Intelligence/index.md` only if a bucket was first-touched
   (rare — buckets exist before consolidation).
8. Append one entry per processed source to `Intelligence/log.md`
   (date, source path, bucket(s) and topic(s) touched, articles
   written/updated, images copied).
9. For each successfully consolidated source file, check the parent
   folder's `README.md` `delete_after_consolidation` flag:
   - `true` (or unset) → **delete the source file and its referenced
     images from `Resources/`**. Only delete after steps 1–8 succeed
     for that file.
   - `false` → leave the source and its images in place. They have
     archival value and are kept as primary documents.
   - On any failure for that file (unreadable, ambiguous bucket,
     write error, image copy error), leave it in place regardless of
     the flag and report it.

**Run buckets in parallel.** One subagent per bucket. Each subagent
enters its bucket, scores the candidate sources, and writes the
articles + topic indexes inside that bucket. The orchestrator reads
each subagent's report and is the only one to touch `_unsorted/`,
`Intelligence/index.md`, and `Intelligence/log.md` (so those files
have a single writer per run).

---

## Verb: `refine`

Read-only audit. **Report findings; do not change anything.** Cover:

- Contradictions between articles within a bucket.
- Orphan articles (no inbound `[[wiki links]]`).
- Concepts referenced but lacking their own article.
- `Intelligence/index.md`, any `_master-index.md`, or any topic
  `_index.md` out of sync with the actual articles/topics on disk.
- Articles that violate the article schema or contain cross-bucket
  wiki links.
- Embedded images missing from the topic folder (broken `![[...]]`
  references).
- Claims marked `(source: needs-verification)` so the user can
  decide whether to chase them.

Run buckets in parallel; one subagent per bucket; one report.

---

## Hard rules

These are non-negotiable. A run that violates any of these is a bug.

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
   `Intelligence/log.md` (always — one append per processed source).
   `Intelligence/index.md` only changes on first-touch of a bucket.
   The `query` verb relies on these indexes being truthful — stale
   indexes silently degrade retrieval, so this is enforced as a hard
   rule, not a courtesy. `refine` flags any drift it finds.
7. **File names are `lowercase-with-hyphens`.** Both folders and
   files. Article slugs match this convention.

---

## Schemas

These are the contract between human-curated and agent-curated layers.
Don't paraphrase them.

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
  consolidated, and **deleted** from `Resources/` once steps 1–8 of
  consolidate succeed. Use for ephemeral material like web clippings.
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

### Citation rules

These apply to every article body. They are part of the schema, not a
suggestion.

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

### Image handling

When a source file references an image (or the article body needs one):

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
consolidate.

---

## On contradictions vs `_unsorted/`

Two failure modes look similar but route differently:

- **No bucket fits** → article goes to `Intelligence/_unsorted/`,
  flagged in the run report, human creates a bucket and re-runs.
- **Sources within a single article disagree** → the article is
  written into its proper bucket/topic with the contradiction noted
  inline per the citation rules. `refine` will list it.

`_unsorted/` is for routing failures, not factual disagreements.
