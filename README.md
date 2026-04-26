# Agentic Persistent Knowledge Management System

A wiki-backed RAG vault operated by an AI agent (Claude Code). Raw sources go in; structured, cited, interlinked knowledge articles come out. The agent handles routing, clustering, and indexing — you define the top-level taxonomy.

---

## Architecture

Two zones, two roles:

```
vault/
├── Resources/          # immutable input — raw sources (clippings, transcriptions, PDFs)
│   └── <folder>/
│       ├── README.md   # controls whether the folder is consolidated and whether sources are deleted after
│       └── *.md / *.pdf / ...
│
└── Intelligence/       # agent-maintained wiki
    ├── index.md        # top-level bucket directory
    ├── log.md          # append-only run log
    ├── _unsorted/      # quarantine for sources that matched no bucket
    └── <bucket>/
        ├── _master-index.md   # bucket scope + topic list
        └── <topic>/
            ├── _index.md      # topic description + article list + related topics
            ├── article-slug.md
            └── image.png      # images live beside their article
```

### Two layers inside `Intelligence/`

| Layer | Who creates it | Rule |
|---|---|---|
| **Buckets** (`Intelligence/<bucket>/`) | Human | Agent never invents a new bucket. Unroutable sources go to `_unsorted/`. |
| **Topics** (`Intelligence/<bucket>/<topic>/`) | Agent | Agent creates topics during `consolidate` when no existing topic fits. |

---

## The three verbs

### `query`

Pull only the relevant context, top-down:

1. Read `Intelligence/index.md` — learn which buckets exist.
2. Pick the best-fit bucket(s) from one-line descriptions.
3. Read the bucket's `_master-index.md` — pick topic(s).
4. Read topic `_index.md` files — pick which article bodies to load.
5. Read only those articles. Follow `[[wiki links]]` within the same bucket only.
6. Return answer with inline citations: `(source: path/to/article.md)`.

**Budget rule:** at most 1 `index.md` + 1–2 `_master-index.md` + a handful of `_index.md` + ≤5 article bodies. Surface to the user if more is needed.

### `consolidate`

Ingest sources from `Resources/` into `Intelligence/`:

1. Walk `Resources/` — skip folders with `include_in_consolidation: false`.
2. Route each source to the best-fit bucket(s) using `_master-index.md` Scope paragraphs.
   - No bucket fits → write to `_unsorted/`, flag in report.
   - Multiple buckets fit → write into each (images copied into each).
3. Within the bucket, pick or create a topic.
4. Write the article (schema below). Copy images into the topic folder.
5. Update `_index.md`, `_master-index.md` (if new topic), `log.md`.
6. Delete source if `delete_after_consolidation: true` (default).

**Parallelism:** run one subagent per bucket. Only the orchestrator writes `_unsorted/`, `index.md`, and `log.md`.

### `refine`

Read-only audit — report only, no writes:

- Contradictions between articles within a bucket.
- Orphan articles (no inbound wiki links).
- Concepts referenced but lacking their own article.
- Index drift (files on disk not in indexes, or vice versa).
- Cross-bucket wiki links (forbidden).
- Broken image embeds.
- Claims marked `(source: needs-verification)`.

---

## Schemas

### `Resources/<folder>/README.md`

```markdown
---
include_in_consolidation: true
delete_after_consolidation: true
---

# Folder name

One paragraph describing what lives here and any handling notes.
```

| Flag | Meaning |
|---|---|
| `include_in_consolidation: false` | Folder skipped entirely during consolidate |
| `delete_after_consolidation: false` | Sources consolidated but originals kept (archival value) |
| `delete_after_consolidation: true` | Sources deleted after successful consolidation (ephemeral material) |
| Missing | Defaults to `true` / `true` |

### `Intelligence/<bucket>/_master-index.md`

```markdown
# Bucket name

**Scope**: One paragraph describing what belongs here and what does not.
This is the routing signal the agent uses when allocating sources.

## Topics

- [[topic-a/_index|Topic A]] — one-line description
- [[topic-b/_index|Topic B]] — one-line description
```

### `Intelligence/<bucket>/<topic>/_index.md`

```markdown
# Topic name

One paragraph describing what this topic clusters.

## Articles

- [[article-slug|Article Title]] — one-line description

## Related Topics

- [[../other-topic/_index|Other Topic]] — one-line note
```

`Related Topics` links same-bucket topics only — never cross-bucket.

### Article

```markdown
# Article title

**Source:** [Source name](url-or-path)
**Author:** if known

---

## Summary

One short paragraph — what this is about and the core thesis.

## Body section

Body content. Every factual claim cites its source inline: `(source: filename.ext)`.

Embed images with `![[image.png]]` — the file must live in the same topic folder.

## Key Takeaways

- ...

## Related

- [[other-article-in-this-bucket]] — one-line note
```

### Citation rules

- Every factual claim gets an inline `(source: filename)` — no bare claims.
- Two sources that disagree: note both inline, mark "unresolved".
- No source available: mark `(source: needs-verification)` so `refine` catches it.

---

## Hard rules

1. **`Resources/` is immutable** during a run. Only post-consolidate deletions are allowed.
2. **Wiki links are same-bucket only.** An article duplicated into two buckets is copied, not linked.
3. **No silent bucket creation.** Unroutable sources go to `_unsorted/`.
4. **Topic creation is allowed and expected.** No existing topic → agent creates one.
5. **Images live beside their article.** No cross-folder image references.
6. **Indexes stay in sync, every run.** Stale indexes silently degrade retrieval — this is enforced, not advisory.
7. **File names are `lowercase-with-hyphens`.**

---

## Getting started

1. Clone this repo (or copy the structure) as your vault root.
2. Create bucket folders under `Intelligence/` with a `_master-index.md` each. Buckets define your macro taxonomy — take time here.
3. Drop source material into `Resources/<folder>/` with a `README.md` declaring the consolidation flags.
4. Point Claude Code at the vault root and tell it to `consolidate`.
5. Ask questions with `query`. Run `refine` periodically to catch drift.

The `CLAUDE.md` at the vault root is the agent's operating manual — it defines the contract verbatim. Keep it in place.

---

## Design notes

- **Progressive disclosure.** The agent walks indexes top-down and reads article bodies only when needed, keeping the context window lean regardless of vault size.
- **Human taxonomy, agent clustering.** Buckets are yours to design. Topics emerge from the material. This split keeps macro structure stable while micro-structure adapts.
- **Self-contained articles.** Images are copied into topic folders on consolidation so source deletion never breaks article embeds.
- **Parallel consolidation.** One subagent per bucket means consolidation scales linearly with bucket count, not vault size.
- **Quarantine over guessing.** An article the agent can't route goes to `_unsorted/` and surfaces in the run report — never silently into a wrong bucket.
