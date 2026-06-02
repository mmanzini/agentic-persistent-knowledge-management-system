# Agentic Knowledge Engine

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
│   ├── context/            # auto-capture drop folder (ephemeral signals, deleted after consolidate)
│   ├── personal/           # prescriptive sources — about-me, writing-rules (kept after consolidate)
│   └── <folder>/
│       ├── README.md       # controls consolidation and post-run deletion
│       └── *.md / *.pdf / ...
│
├── Skills/                 # agent skill definitions (read-only during vault verbs)
│   └── auto-capture/
│       └── SKILL.md        # auto-capture skill — writes signals to Resources/context/
│
└── Intelligence/           # agent-maintained wiki
    ├── index.md            # top-level bucket directory
    ├── log.tsv             # append-only run log (consolidate / refine / evaluate rows)
    ├── _unsorted/          # quarantine for sources that matched no bucket
    ├── _eval/
    │   ├── questions.md    # fixed evaluation question set
    │   └── results.tsv     # per-question results across evaluate runs
    ├── _episodes/          # episodic memory — the agent's experiences (see below)
    │   ├── _index.md       # thin router: the three kinds + tag vocabulary
    │   ├── operational/    # the agent's own verb runs
    │   ├── life/           # derived from Resources/Daily/ digests
    │   ├── signals/        # derived from Resources/context/ auto-capture drops
    │   └── reflections.md  # distilled, merged generalized patterns
    └── <bucket>/
        ├── _master-index.md   # bucket scope + topic list
        └── <topic>/
            ├── _index.md      # topic description + article list + related topics
            ├── article-slug.md
            └── image.png      # images live beside their article
```

### Two kinds of memory

`Intelligence/` holds **semantic** memory — the buckets are facts:
cited, indexed, interlinked articles. `Intelligence/_episodes/` holds
**episodic** memory — the agent's record of *experiences* (goal →
actions → outcome → insight): its own verb runs, plus date-keyed life
episodes and captured signals. Semantic memory answers *"what is
true?"*; episodic memory answers *"what happened, and what worked last
time?"* The agent **recalls** relevant episodes into context at the
start of each run, so curation and retrieval compound instead of
restarting from scratch every time — the autoresearch loop applied to
the agent's own behaviour.

### Two layers inside `Intelligence/`

| Layer | Who creates it | Rule |
|---|---|---|
| **Buckets** (`Intelligence/<bucket>/`) | Human | Agent never invents a new bucket. Unroutable sources go to `_unsorted/`. |
| **Topics** (`Intelligence/<bucket>/<topic>/`) | Agent | Agent creates topics during `consolidate` when no existing topic fits. |

### `schema.md` — the frozen harness

`schema.md` at the vault root is the machine-readable schema contract. The agent reads it once at session start and treats it as fixed. **No verb may modify it.** It duplicates the hard rules and file schemas verbatim. If a verb run requires a schema change to succeed, that is a signal to stop and surface the question to the human — not to edit the file.

---

## The verbs

Four core verbs — `query`, `consolidate`, `refine`, `evaluate` — plus
`reflect`, which maintains episodic memory. **Every run is recall → act
→ capture:** the agent recalls relevant past episodes before acting and
writes a new episode after, so experience accumulates across runs.

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

**Per-bucket workers:** one worker per bucket — run in parallel if the runtime supports it, otherwise sequentially. Only the orchestrator writes `_unsorted/`, `index.md`, `log.tsv`, and `_eval/results.tsv`.

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

Episodic recall has its own cost line: `recall_reads` (episode bodies read during recall) is tracked **separately** from `files_read`, so the wiki-retrieval baseline stays comparable. The `eval_summary` row also trends `quarantine_rate` — together these show whether learning from experience is actually paying off (faster routing, fewer mis-routes) over time.

### `reflect`

Maintains episodic memory — reads only `_episodes/`, writes only inside it:

1. Distills recurring `Insights` across episodes into `reflections.md` — **merging and rewriting**, never appending unboundedly, so the file stays small as the store grows.
2. Stamps folded episodes `distilled: true` so they leave the hot recall surface (kept on disk for audit).
3. Appends a `reflect_summary` row to `log.tsv`.

`reflect` **auto-chains** after `consolidate` (→ `refine` → `reflect`). It never edits `CLAUDE.md` or `schema.md` — reflections feed run *context* only, never the program.

**Why recall stays cheap as episodes pile up:** a recall walk reads the `_episodes/` router, one kind-index of one-liners, then at most **3 episode bodies** plus the small merged `reflections.md`. `distilled` episodes are skipped. So recall cost is bounded — it does not grow with the store.

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
| Episode integrity | Episodes missing required frontmatter/sections, or any `[[ ]]` link inside an episode that points outside `_episodes/` (episodes reference articles by `(source: …)` citation only) |

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
8. **Single locus of change.** Per-bucket workers write only inside their assigned subtree. The orchestrator is the sole writer of `index.md`, `log.tsv`, `_unsorted/`, `_eval/results.tsv`, and `_episodes/`.
9. **Schema is frozen.** No verb may modify `schema.md`.

---

## Getting started

1. Clone this repo as your vault root.
2. Replace the placeholder `domain-a` / `domain-b` buckets with your own macro taxonomy. Each bucket needs a `_master-index.md` with a Scope paragraph.
3. Replace or extend `_eval/questions.md` with questions your vault should be able to answer. These are your benchmark — be specific.
4. Fill in `Resources/personal/about-me.md` and `Resources/personal/writing-rules.md` with your own identity and style notes. These are prescriptive — the agent preserves your wording rather than paraphrasing.
5. Drop source material into `Resources/<folder>/` with a `README.md` declaring the consolidation flags.
6. Point Claude Code at the vault root and run `consolidate`.
7. Ask questions with `query`. Run `refine` periodically to catch drift. Run `evaluate` to track retrieval quality over time.

The `CLAUDE.md` and `schema.md` at the vault root are the agent's operating contract — keep them in place.

### Auto-capture

The `Skills/auto-capture/SKILL.md` skill lets the agent quietly persist conversational signals (decisions, preferences, opinions, facts) to `Resources/context/` during any conversation — no explicit "save this" needed. Those captures accumulate and are consolidated into `Intelligence/` on the next `consolidate` run, then deleted per the folder's `delete_after_consolidation: true` flag.

---

## Design notes

- **Progressive disclosure.** The agent walks indexes top-down and reads article bodies only when needed, keeping the context window lean regardless of vault size. **Episodic recall obeys the same discipline:** a bounded tag-matched walk (≤3 episode bodies) plus a `reflections.md` that *compresses* many episodes into a few generalized lines, so recall cost stays roughly constant even as the episode store grows.
- **Two kinds of memory.** Semantic memory (buckets = facts) answers "what is true?"; episodic memory (`_episodes/` = experiences) answers "what happened, and what worked last time?" Recalling episodes at run-start lets curation and retrieval compound across runs instead of restarting cold — and `reflect` distils experience into reusable heuristics without ever touching the frozen `CLAUDE.md` / `schema.md` bedrock.
- **Human taxonomy, agent clustering.** Buckets are yours to design. Topics emerge from the material. This split keeps macro structure stable while micro-structure adapts.
- **Self-contained articles.** Images are copied into topic folders on consolidation so source deletion never breaks article embeds.
- **Per-bucket workers.** One worker per bucket — parallel where the runtime supports it, sequential otherwise — means consolidation scales with bucket count, not vault size.
- **Quarantine over guessing.** An article the agent can't route goes to `_unsorted/` and surfaces in the run report — never silently into a wrong bucket.
- **Measurable quality.** The `evaluate` verb gives the wiki a fixed benchmark. `files_read` is the primary signal: a well-structured wiki should answer questions in fewer reads as consolidation matures.
