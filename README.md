# Agentic Persistent Knowledge Management System

A wiki-backed RAG vault operated by an AI agent (Claude Code). Raw sources go in; structured, cited, interlinked knowledge articles come out. The agent handles routing, clustering, and indexing — you define the top-level taxonomy.

---

## Architecture

Two zones, two roles:

```
vault/
├── CLAUDE.md               # agent operating contract (verbs, rules, schemas)
├── schema.md               # frozen harness — read-only to the agent
│
├── Resources/              # immutable input — raw sources (clippings, transcriptions, PDFs)
│   └── <folder>/
│       ├── README.md       # controls consolidation and post-run deletion
│       └── *.md / *.pdf / ...
│
└── Intelligence/           # agent-maintained wiki
    ├── index.md            # top-level bucket directory
    ├── log.tsv             # append-only run log (consolidate / refine / evaluate rows)
    ├── _unsorted/          # quarantine for sources that matched no bucket
    ├── _eval/
    │   ├── questions.md    # fixed evaluation question set
    │   └── results.tsv     # per-question results across evaluate runs
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

### `schema.md` — the frozen harness

`schema.md` at the vault root is the machine-readable schema contract. The agent reads it once at session start and treats it as fixed. **No verb may modify it.** It duplicates the hard rules and file schemas verbatim. If a verb run requires a schema change to succeed, that is a signal to stop and surface the question to the human — not to edit the file.

---

## The four verbs

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
5. Update `_index.md`, `_master-index.md` (if new topic), `log.tsv`.
6. Delete source if `delete_after_consolidation: true` (default).

**Parallelism:** one subagent per bucket. Only the orchestrator writes `_unsorted/`, `index.md`, `log.tsv`, and `_eval/results.tsv`.

### `refine`

Read-only audit — report only, no writes:

- Contradictions between articles within a bucket.
- Orphan articles (no inbound wiki links from other articles).
- Concepts referenced but lacking their own article.
- Index drift (files on disk not in indexes, or vice versa).
- Cross-bucket wiki links (forbidden).
- Broken image embeds.
- Claims marked `(source: needs-verification)`.

Appends one `refine_summary` row to `log.tsv` with a **drift count** (see below).

### `evaluate`

Runs the fixed question set in `_eval/questions.md` against the live wiki. For each question the agent walks the indexes, reads articles, and self-scores:

- `files_read` — number of article bodies read to answer (the primary metric — lower is better as the wiki matures).
- `answer_quality` — `good` | `partial` | `poor` | `missing`.

Results append to `_eval/results.tsv`. The orchestrator also writes one `eval_summary` row to `log.tsv` with aggregate counts. Running `evaluate` regularly lets you track whether `consolidate` is making the wiki better at answering the questions you actually care about.

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
- Two sources that disagree: note both inline, mark "Unresolved".
- No source available: mark `(source: needs-verification)` so `refine` catches it.

---

## Log and eval formats

### `Intelligence/log.tsv`

Tab-separated, 9 columns. One row per processed source for `consolidate`; one summary row per run for `refine` and `evaluate`.

```
timestamp  verb  source  bucket  topic  articles_changed  images_copied  status  notes
```

- `status` — `kept` | `quarantined` | `crashed` | `refine_summary` | `eval_summary`
- `refine_summary` rows must include `drift=<N>` in `notes`.
- `eval_summary` rows must include `total_files_read=<N>` and `quality=<Ng/Np/Npoor/Nm>` in `notes`.

### `Intelligence/_eval/results.tsv`

Tab-separated, 5 columns. One row per question per `evaluate` run.

```
timestamp  question_id  files_read  answer_quality  notes
```

- `files_read` — article bodies read (excludes index reads). The primary metric. Lower is better.
- `answer_quality` — `good` | `partial` | `poor` | `missing`.

---

## Drift count

`refine` produces a single integer — the sum of:

| Component | What it counts |
|---|---|
| Orphans | Articles with zero inbound `[[wiki links]]` from other articles in their bucket |
| Index mismatches | Entries in any index that don't resolve to a real file, or files on disk not listed |
| Broken embeds | `![[image.ext]]` references missing from the topic folder |
| Cross-bucket links | Any `[[link]]` that crosses bucket boundaries |
| Schema violations | Articles missing required sections or factual claims without citations |

The orchestrator compares the current drift count to the previous `refine_summary` row and reports **advance** (drift down, or flat with `articles_changed > 0`), **hold** (flat, no changes), or **regress** (drift up). Regressions are surfaced — never silently undone.

---

## Hard rules

1. **`Resources/` is immutable** during a run. Only post-consolidate deletions are allowed.
2. **Wiki links are same-bucket only.** An article duplicated into two buckets is copied, not linked.
3. **No silent bucket creation.** Unroutable sources go to `_unsorted/`.
4. **Topic creation is allowed and expected.** No existing topic → agent creates one.
5. **Images live beside their article.** No cross-folder image references.
6. **Indexes and log stay in sync, every run.** Stale indexes silently degrade retrieval — enforced, not advisory.
7. **File names are `lowercase-with-hyphens`.**
8. **Single locus of change.** Per-bucket subagents write only inside their assigned subtree. The orchestrator is the sole writer of `index.md`, `log.tsv`, `_unsorted/`, and `_eval/results.tsv`.
9. **Schema is frozen.** No verb may modify `schema.md`.

---

## Getting started

1. Clone this repo as your vault root.
2. Replace the placeholder `domain-a` / `domain-b` buckets with your own macro taxonomy. Each bucket needs a `_master-index.md` with a Scope paragraph.
3. Replace or extend `_eval/questions.md` with questions your vault should be able to answer. These are your benchmark — be specific.
4. Drop source material into `Resources/<folder>/` with a `README.md` declaring the consolidation flags.
5. Point Claude Code at the vault root and run `consolidate`.
6. Ask questions with `query`. Run `refine` periodically to catch drift. Run `evaluate` to track retrieval quality over time.

The `CLAUDE.md` and `schema.md` at the vault root are the agent's operating contract — keep them in place.

---

## Design notes

- **Progressive disclosure.** The agent walks indexes top-down and reads article bodies only when needed, keeping the context window lean regardless of vault size.
- **Human taxonomy, agent clustering.** Buckets are yours to design. Topics emerge from the material. This split keeps macro structure stable while micro-structure adapts.
- **Self-contained articles.** Images are copied into topic folders on consolidation so source deletion never breaks article embeds.
- **Parallel consolidation.** One subagent per bucket means consolidation scales with bucket count, not vault size.
- **Quarantine over guessing.** An article the agent can't route goes to `_unsorted/` and surfaces in the run report — never silently into a wrong bucket.
- **Measurable quality.** The `evaluate` verb gives the wiki a fixed benchmark. `files_read` is the primary signal: a well-structured wiki should answer questions in fewer reads as consolidation matures.
