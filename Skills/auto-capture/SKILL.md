---
name: auto-capture
description: >
  Use this skill whenever the user expresses something that adds signal to their evolving
  intelligence layer — decisions, priority shifts, preferences, stated opinions, reactions
  strong enough to be preferences, writing-style or tone-of-voice observations, non-trivial
  new facts about projects, people, tools, or topics they care about. The skill writes the
  signal to a configured drop folder (default: `Resources/context/` at the vault root) as a
  single markdown file, then continues the conversation. Capture liberally; consolidate
  decides what's worth keeping. Do NOT use for purely time-bounded task details ("try X
  next", "remind me in this conversation"), polite-conversation noise, or to restate signals
  already covered by a recent capture.
---

# Auto-capture

Quietly persist conversational signal so the user's intelligence layer stays fresh without explicit "save this" requests. The skill is a writer into the `Resources/` layer of a vault; the vault's normal `consolidate` verb handles routing into `Intelligence/`.

## When to fire

Use this skill in the background of any conversation when the user expresses one of these:

- **Decisions with rationale** — "I'm going to deprioritise X because Y."
- **Priority shifts** — "X is on hold for now; Y is the focus."
- **Preferences** (working style, tools, processes) — "I prefer guided flows over blank slates."
- **Stated opinions or strong reactions** — "I find always-loaded context dishonest." "Auto-genericising names drives me crazy."
- **Writing-style or tone-of-voice observations** — "I dislike em-dash overuse." "Sentence-case headings, always."
- **Non-trivial new facts** about projects, people, tools, or topics the user cares about — "The vault now lives under `Documents/vault`." "My manager is moving teams."

Capture **liberally**. The downstream `consolidate` step deduplicates and merges; this skill should err toward saving more rather than less. A capture that turns out redundant costs nothing; a capture missed is gone.

## When NOT to fire

- **Throwaway task talk.** "Try the other regex first" / "open that file again". Time-bounded to the current task, no durable signal.
- **Polite-conversation noise.** "Thanks", "got it", "yeah that works".
- **Restating signals already captured this session.** If the user reaffirms something the skill already wrote a file for, don't write another. Append to the existing file only if the user explicitly amended it.
- **Active-task narration the user is dictating.** If they're talking through what an agent should do *right now*, that's task content, not durable signal.

## What to write

One markdown file per captured signal. No batching, no overwriting, no editing existing capture files (with the one amendment exception above).

**Path:** `<destination_folder>/YYYY-MM-DD-<slug>.md` where:
- `YYYY-MM-DD` is today's date (UTC or local — pick one and be consistent).
- `<slug>` is a short kebab-case summary of the signal (≤ 6 words). Examples: `project-x-deferred`, `prefers-guided-flows`, `dislikes-em-dash-overuse`, `vault-relocated`.

If the same date+slug already exists, append a numeric suffix: `YYYY-MM-DD-<slug>-2.md`.

**File body:**

```markdown
# <One-line summary of the signal>

**Captured:** YYYY-MM-DD HH:MM (timezone)
**Origin:** conversation

<2–6 sentences capturing the signal in the user's voice as best as possible. Quote the user directly when the wording matters (decisions, opinions, preferences). Add minimal context for what prompted it, but don't editorialise — keep paraphrasing light. The downstream `consolidate` step will turn this into a properly-cited wiki article; this file just needs to preserve the signal faithfully.>
```

No frontmatter. No tags. No links. The file is a primary source — keep it raw.

## Confirmation back to the user

After writing, confirm in **one line** without breaking conversation flow:

> Captured to `Resources/context/2026-04-29-prefers-guided-flows.md`.

That's it. Don't summarise what you captured back to them; they just said it. Don't ask if it was right — they can edit the file. Don't suggest related captures.

## Configuration

The destination folder is the only configurable part of the skill. Read it in this order:

1. **`AUTO_CAPTURE_DEST` environment variable**, if set.
2. **`.auto-capture-config.json`** in the current working directory, with shape `{ "destination": "<absolute or vault-relative path>" }`.
3. **Default**: `Resources/context/` relative to the vault root. Detect the vault root by walking upward from the current working directory until you find a directory that contains both `schema.md` and an `Intelligence/` folder as siblings.
4. **If no vault is found and no config is present**, ask the user once where to put captures, then offer to write a `.auto-capture-config.json` so the question doesn't repeat.

## Vault-aware behaviour

When operating inside a vault:

- Captures go to `Resources/context/`. They get deleted on the next `consolidate` per that folder's `delete_after_consolidation: true`. This is correct: their value transfers to `Intelligence/`.
- If the user wants a capture preserved as a permanent source, **suggest moving it** to `Resources/personal/` in your one-line confirmation. Don't move it for them — that's the user's call.

When operating outside a vault, write to whatever destination the config specifies. The skill is a writer; lifecycle is the vault's problem.

## What this skill does NOT do

- **Does not write into `Intelligence/`.** Ever. That's `consolidate`'s job.
- **Does not run `consolidate`.** Captures accumulate; the user runs `consolidate` when they want to drain them.
- **Does not deduplicate.** If the user expresses the same preference twice in a session, that's two files. `consolidate` and `refine` deduplicate downstream.
- **Does not edit `about-me.md` or `writing-rules.md`.** Those are static, hand-curated. The skill only writes new files.
