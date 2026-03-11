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

## Empirical Validation

Evaluated on LongMemEval-S (4,575 sessions, 100 questions).
Controlled comparison: C6 (dual-trace) vs C7 (fact-only), identical coverage.

  Category            C7 (fact-only)   C6 (dual-trace)   Scene contribution
  --------------------|----------------|-----------------|------------------
  Overall             54%              73.7%             +19.7pp
  Single-session      79%              79%                 0pp (null -- expected)
  Multi-session       43%              65.5%             +22.2pp
  Knowledge-update    59%              81.8%             +22.7pp
  Temporal-reasoning  38%              70.8%             +33.3pp
  Token cost          baseline         -3.3%             cheaper per query

Full performance ladder:

  Vanilla (no memory)    ~9%
  Basic archival         48%
  Fact-only (C7)         54%
  Dual-trace (C6)        73.7%    <- this skill
  SOTA                   84-86%

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
