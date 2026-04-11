# Drawing-Memory Skill: Letta Code / Context Repository Edition

Dual-trace memory encoding for Letta Code agents with memfs enabled.
Inspired by the drawing effect (Fernandes et al., 2018).

This is the Letta Code / context repository version. For the Letta V1 API
agent version (archival_memory_insert), see:
https://github.com/sternb12/agent_draw_skills

---

## The Key Difference

  Letta V1 API agents:   store -> archival_memory_insert (vector DB)
                         retrieve -> archival_memory_search (cosine top-k)

  Letta Code agents:     store -> Write to memory/facts/ + memory/scenes/
                         retrieve -> Read + Glob (exact lookup, full enum)

File-based storage solves multi-session aggregation structurally:
Glob("memory/facts/*.md") enumerates everything. No symbolic index needed.

---

## Parent Evaluation (letta_v1_agent / archival memory condition)

The dual-trace encoding principle was evaluated on LongMemEval-S (Wang et al.,
2024): 4,575 real user conversation sessions, 100 structured recall questions,
4,575-session distractor corpus.

Controlled comparison: C6 (dual-trace) vs C7 (fact-only). Paired analysis over
99 shared questions, bootstrap CIs (10,000 resamples), GPT-4o graded. Both
conditions used the letta_v1_agent architecture with archival_memory_insert and
archival_memory_search -- the vector-DB variant, not the file-based format this
repo implements. Session encoding rates were comparable: C6 54.8%, C7 57.4%.

  Category            C7 (fact-only)   C6 (dual-trace)   Scene contribution
  --------------------|----------------|-----------------|------------------
  Overall             53.5%            73.7%             +20.2pp
  Single-session      75%              75%                 0pp (null -- expected)
  Multi-session       20%              50%               +30pp
  Knowledge-update    55%              80%               +25pp
  Temporal-reasoning  25%              65%               +40pp
  Token cost          baseline         -3.3%             cheaper per query

Full performance ladder (letta_v1_agent condition):

  Vanilla (no memory)    20.0%    (correct abstention, no stored sessions)
  Basic archival         47.5%
  Fact-only (C7)         53.5%
  Dual-trace (C6)        73.7%
  SOTA                   84-86%

Full results, reproducibility details, and evidence-scoring docs:
https://github.com/sternb12/agent_draw_skills

---

## This Variant: Architectural Adaptation

This file-based variant is an architectural adaptation of the evaluated protocol.
Storage moves from archival_memory_insert (vector DB) to Write + Glob (flat
files in the agent's git-backed memory filesystem). Retrieval moves from
archival_memory_search (probabilistic cosine top-k) to Glob + Read (complete
enumeration -- solving multi-session aggregation structurally, no symbolic
index needed).

The dual-trace encoding structure -- fact file paired with scene file, shared
anchor, same temporal anchor rules -- is preserved from the evaluated protocol.

Validation status: preliminary pilot validation on a Letta Code agent (four
manual tests, March 2026). Controlled experimental evaluation against
LongMemEval-S or a comparable benchmark is future work.

---

## Installation

1. Clone into .skills/ in your Letta Code project:

     git clone https://github.com/sternb12/letta-code-draw-skill .skills/drawing-memory

2. Enable memfs for your agent (/memfs enable)

3. Run /init to bootstrap the agent's memory structure

4. The skill auto-discovers on startup. The agent will load it when a
   user shares information worth remembering.

---

## How It Works

Encoding (when user shares memorable information):
  1. Score evidence (0-12)
  2. Write memory/facts/{anchor}.md -- structured paraphrase
  3. Write memory/scenes/{anchor}.md -- visual scene with temporal anchor
  4. Commit to memfs git

Retrieval (when user asks a memory question):
  - State A: fact + scene found -> reconstruct scene, answer with high confidence
  - State B: fact only -> answer with medium confidence
  - State C: no file found -> abstain, do not confabulate
  - Aggregate: Glob all facts/, collect all matches, sequence via scene anchors

---

## Research Citation

Fernandes, M. A., Wammes, J. D., & Meade, M. E. (2018). The surprisingly
powerful influence of drawing on memory. Current Directions in Psychological
Science, 27(5), 302-308. https://doi.org/10.1177/0963721418755385
