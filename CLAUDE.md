# Vault — the iterable program

This vault is one wiki-backed RAG, structured per the autoresearch
pattern: a **frozen harness** in [schema.md](schema.md) and an
**iterable program** here. Iterate on this file over time to
improve agent behaviour; never edit `schema.md` mid-run.

## Zones

- **`Resources/`** — immutable input. Raw source material. The agent
  reads from here and may delete sources after consolidating them
  (see the README flag), but otherwise never modifies this tree.
- **`Intelligence/`** — the wiki. Long-term memory, agent-maintained.
  Two layers inside it:
  - **Buckets** (`Intelligence/<bucket>/`) — **user-curated** macro
    groups. Each has a `_master-index.md` declaring its scope. The
    agent **never** invents a new bucket; quarantine to `_unsorted/`
    instead.
  - **Topics** (`Intelligence/<bucket>/<topic>/`) — **agent-curated**
    clusters created during `consolidate`. Topic creation is allowed
    and expected.
- **`Skills/`** — agent skill definitions. Read-only for vault verbs
  (`query`, `consolidate`, `refine`, `evaluate`).

## Session start

1. Read [schema.md](schema.md) once. Treat it as fixed for the
   session. Do not edit it.
2. Read this file (the verbs and orchestration rules below).
3. Read `Intelligence/index.md` only when entering the wiki for a
   `query` or `consolidate`.

## Verbs

The vault has four verbs: **`query`** (read), **`consolidate`**
(write from `Resources/`), **`refine`** (audit, read-only), and
**`evaluate`** (measure). Schemas, hard rules, log/eval formats, and
the drift-count definition all live in `schema.md`.

---

### `query`

The point of the wiki. Walk the indexes top-down; read article
bodies last.

1. **Start at `Intelligence/index.md`.** Learn which buckets exist.
2. **Pick the bucket(s).** Match against the one-line bucket
   descriptions and, if needed, the bucket's `_master-index.md`
   Scope paragraph. If no bucket fits, say so and stop. Don't
   guess into `_unsorted/` — it's a quarantine, not a search target
   unless the user explicitly asks.
3. **Pick the topic(s) within the chosen bucket(s).** Read the
   bucket's `_master-index.md` Topics list and, if disambiguation is
   needed, the candidate topics' `_index.md` description paragraphs.
   **Do not load article bodies yet.**
4. **Read articles.** Within the chosen topic(s), use the topic
   `_index.md` Articles list (each article has a one-line
   description) to pick which article files to actually `Read`.
   Pull only those. The topic's `Related Topics` block tells you
   where to step sideways within the same bucket.
5. **Stop at bucket boundaries.** Wiki links inside articles are
   guaranteed same-bucket only, so following them during query is
   safe. If a question genuinely spans buckets, repeat the process
   per bucket and return parallel answers — never assume
   cross-bucket continuity.
6. **Cite back.** Every fact in the answer cites the article path
   it came from, in the same `(source: <path>)` form articles use
   internally.
7. **Budget.** Default ceiling: one `index.md` + one or two
   `_master-index.md` + a handful of `_index.md` + the article
   bodies actually needed. If a question seems to require more
   than 5 article bodies, surface that to the user before
   continuing — usually the question is too broad or the buckets
   are mis-cut.

**Personal context auto-pull.** `query`-style walks aren't only
triggered by explicit user questions. When the task at hand is
**generative and personal-context-relevant** — drafting prose,
writing a post or an email, designing user-facing copy, deciding
tone, or making a choice where the user's preferences matter — the
agent walks `Intelligence/personal/` *before generating*, if that
bucket exists. The trigger is the task type, not the user's wording.
For purely technical or reference work (debugging, code reading,
"how does library X behave"), skip the personal bucket entirely.
The bucket's `_master-index.md` Scope paragraph is the routing
signal — read it first if unsure.

---

### `consolidate`

When the user says "consolidate":

1. Walk `Resources/` recursively. For each subfolder, read its
   `README.md` and skip if `include_in_consolidation: false`.
2. For every remaining source file, pick the best-fit **bucket(s)**
   by comparing content against each
   `Intelligence/<bucket>/_master-index.md` Scope paragraph.
   - No bucket fits → write the article into
     `Intelligence/_unsorted/`, append a row with
     `status=quarantined` to `log.tsv`, and flag it in the run
     report. **Never silently invent a bucket.**
   - Two or more buckets fit → write the article into each, with
     images copied into each per `schema.md` *Image handling*.
3. Within the chosen bucket, pick the best-fit **topic** by
   comparing content against each `<topic>/_index.md` description.
   No topic fits → **create a new topic folder** with its
   `_index.md` and add it to the bucket's `_master-index.md`.
4. Write or update the article using the *Article schema* from
   `schema.md`. Copy any referenced images into the topic folder.
5. Update the topic's `_index.md` (add the article, refresh
   `Related Topics` if needed).
6. Update the bucket's `_master-index.md` if a new topic was
   created.
7. Update `Intelligence/index.md` only if a bucket was first-touched
   (rare).
8. Append one row per processed source to `Intelligence/log.tsv`
   in the format defined in `schema.md`. The orchestrator is the
   sole writer of `log.tsv` (subagents emit row data; the
   orchestrator appends).
9. For each successfully consolidated source, honour
   `delete_after_consolidation`:
   - `true` (or unset) → delete the source file and its referenced
     images from `Resources/`. Only after steps 1–8 succeed.
   - `false` → leave in place.
   - On any failure for that file, leave in place regardless and
     log `status=crashed` with the error in `notes`.
10. **Auto-chain `refine`.** After consolidate finishes, immediately
    run `refine` (see below) and surface its drift-count delta vs
    the previous `refine_summary` row in the same run report.

**Parallelism.** One subagent per bucket. Each subagent enters its
bucket, scores candidate sources, and writes articles + topic
indexes inside that bucket only. Subagents return row data and a
list of touched paths to the orchestrator. The orchestrator is the
sole writer of `Intelligence/index.md`, `Intelligence/log.tsv`,
`Intelligence/_unsorted/`, and `Intelligence/_eval/results.tsv` —
this is hard rule 8 in `schema.md`.

**Prescriptive sources (`Resources/personal/`).** Sources in
`Resources/personal/` (e.g. `about-me.md`, `writing-rules.md`) are
prescriptive — identity statements and writing rules whose voice
*is* the value. When deriving articles from them, **preserve the
original wording**. Quote or near-verbatim copy with light
structural framing rather than paraphrase. Cite the source normally
with `(source: <filename>)`. Paraphrasing destroys the point.
This applies only to `Resources/personal/`; sources in
`Resources/context/` and elsewhere follow the normal summarise-and-
cite flow.

---

### `refine`

Read-only audit. **Report findings; do not change anything.** Per
bucket, in parallel; one orchestrator-written summary row.

Cover (these are the components of the **drift count** — see
`schema.md`):

- **Orphan articles** — zero inbound `[[wiki links]]` from other
  articles in the bucket.
- **Index mismatches** — `_index.md`, `_master-index.md`, or
  `index.md` entries that don't resolve, or files on disk not
  listed in their parent index.
- **Broken `![[...]]` embeds** — image references that don't
  resolve in the same topic folder.
- **Cross-bucket wiki links** — hard rule 2 violations.
- **Schema violations** — articles missing required sections, or
  factual claims without `(source: ...)` citations.

Also report (do not count toward drift; track separately in `notes`):

- **Contradictions between articles** within a bucket.
- **Concepts referenced but lacking their own article.**
- **Claims marked `(source: needs-verification)`** so the user can
  decide whether to chase them.
- **Simplification candidates** — pairs of thin topics that could
  merge with no information loss, or articles redundant enough to
  prune. Propose, do not apply. Apply the autoresearch simplicity
  criterion: *equal information content + simpler structure = win*.

Append one summary row per run to `log.tsv` with
`status=refine_summary`, `notes` including
`drift=<N>` plus a brief breakdown.

**Advance-on-improvement.** The orchestrator compares this run's
drift to the previous `refine_summary` row and labels the run
**advance** / **hold** / **regress** in the user-facing report.
Never silently undo work — surface the regression and let the human
decide.

---

### `evaluate`

The wiki's **val_bpb analog**. Run `query` against each question in
`Intelligence/_eval/questions.md`, log read-cost and answer quality
to `_eval/results.tsv`, and produce a summary.

1. Read `Intelligence/_eval/questions.md` and parse the question
   list (each has an ID, prompt, and target hint).
2. For each question:
   a. Run the `query` verb on the prompt. **Count `files_read` as
      article-body reads only** — do not count `index.md`,
      `_master-index.md`, or `_index.md` reads. Those are routing
      overhead, not retrieval cost.
   b. Self-judge `answer_quality` against the question's intent:
      - `good` — answer is complete and well-cited.
      - `partial` — answer is on-target but missing detail or
        forced to lean on a `(source: needs-verification)` claim.
      - `poor` — answer is vague, mis-routed, or contradicts the
        sources.
      - `missing` — no relevant article exists; the wiki can't
        answer this yet.
   c. Append a row to `Intelligence/_eval/results.tsv` per the
      schema in `schema.md`.
3. Append a single `status=eval_summary` row to `log.tsv` with
   `notes` including `total_files_read=<N>` and a quality breakdown
   (e.g. `quality=2g/2p/1m`).
4. Compare to the previous `eval_summary` row (if any) and label
   the run **advance** (total_files_read down, or quality up with
   files_read flat), **hold**, or **regress**.

`evaluate` is read-only against the wiki — the only writes are to
`log.tsv` and `_eval/results.tsv`, both orchestrator-written.

---

## Orchestration rules

- **One orchestrator per run.** Spawns the per-bucket subagents,
  collects their reports, and is the **sole writer** of
  `Intelligence/index.md`, `Intelligence/log.tsv`,
  `Intelligence/_unsorted/`, and `Intelligence/_eval/results.tsv`.
- **Subagent scope.** A subagent assigned to bucket `X` writes
  **only** inside `Intelligence/X/`. Any attempt to write outside
  is a hard-rule-8 violation and aborts the subagent.
- **Auto-chain.** `consolidate` → `refine` always. `evaluate` runs
  on user request and is recommended after any structurally
  significant `consolidate` (new bucket touched, >3 articles
  written) so the read-cost trajectory stays visible.
- **Run summary format.** Every run report ends with one line:
  `run=<verb> advance|hold|regress drift=<N> Δdrift=<±N>` (for
  `refine`/`consolidate`) or `total_files_read=<N> quality=<...>`
  (for `evaluate`).

## On contradictions vs `_unsorted/`

Two failure modes look similar but route differently:

- **No bucket fits** → article goes to `Intelligence/_unsorted/`,
  `status=quarantined`, human creates a bucket and re-runs.
- **Sources within a single article disagree** → article is
  written into its proper bucket/topic with the contradiction noted
  inline per the citation rules. `refine` lists it.

`_unsorted/` is for routing failures, not factual disagreements.

## How to iterate on this file

This file is `program.md`-shaped: edit it to improve agent
behaviour. Examples of changes that belong here (not in `schema.md`):

- Tightening the bucket-routing heuristic in `consolidate` step 2.
- Adjusting the read-budget ceiling in `query` step 7.
- Adding new orchestration rules.
- Adding a new verb (e.g. `summarize` for whole-bucket digests).

Examples of changes that belong in `schema.md` (require deliberate
human commit):

- Changing the article schema or citation format.
- Adding/removing a column in `log.tsv` or `_eval/results.tsv`.
- Changing what counts toward the drift count.
- Changing the hard rules.

If you find yourself wanting to edit `schema.md` to make a verb
work, stop and surface the question — the harness is supposed to be
stable.
