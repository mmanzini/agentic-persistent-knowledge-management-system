# Build agents to learn from experiences using Amazon Bedrock AgentCore episodic memory

**Source:** https://aws.amazon.com/blogs/machine-learning/build-agents-to-learn-from-experiences-using-amazon-bedrock-agentcore-episodic-memory/
**Author:** AWS Machine Learning Blog
**Captured:** 2026-06-02

Source material for the episodic-memory idea (agentic-knowledge-engine project; article 003 in articles-and-essays).

## Episode design

Treat an "episode" as an atomic experience:

`<Goal> → <Actions Taken> → <Outcome/Reflections>`

## Episodic memory strategy — the pipeline

The strategy sits between short-term and long-term memory. The agentic
application writes to short-term memory; the episodic strategy turns raw
turns into durable, retrievable experience, then writes to long-term
memory the agent reads back.

Three stages:

1. **Turn extraction** — analyses messages to determine episode
   completion (where one experience starts and ends).
2. **Episode extraction** — organises the extracted information into
   structured episodes.
3. **Reflection** — analyses patterns *across* episodes to help the
   agent learn and make better decisions.

(Diagram 1 — high-level flow: Agentic application → Short-term memory →
[Episodic memory strategy: Turn extraction → Episode extraction →
Reflection] → Long-term memory → Agentic application. Reflection and
episodes "Write to" long-term memory.)

## Implementation detail

(Diagram 2 — AgentCore implementation: Agentic application —*Send
events*→ Short-term memory —*Async*→ batches of N messages → Turn
extraction (*prompt to extract episode turns*) → Episode extraction
(*prompt to extract and evaluate episodes*) → Extracted episodes →
AgentCore memory vector store —*Retrieve memories*→ Agentic
application.)

Key additions over the high-level view:

- Processing is **async** and **batched** (N messages at a time), not
  inline with the conversation.
- **Cross-episode reflection** takes newly extracted episodes plus the
  **K most similar episodes** retrieved from the vector store, runs a
  *prompt to reflect*, and produces a **reflection memory** written
  back into the store.
- **Amazon Bedrock** runs the extraction, evaluation, and reflection
  prompts.
- Episodes and reflections both land in the **AgentCore memory vector
  store**, which the agent queries to retrieve memories at run time.

## Relevance to the knowledge engine

The vault is a semantic/structured RAG layer. Episodic memory adds a
complementary event-based layer: time-stamped <goal → actions →
outcome> records, with a reflection pass that mines patterns across
episodes. Open questions for the idea: where episodic records live (new
zone vs an extension of `log.tsv`), how reflection consolidates into the
wiki, and how `query` arbitrates between episodic and semantic recall.
