---
name: drawing-memory
description: >
  Dual-trace memory encoding for Letta Code agents with context repositories.
  Inspired by the drawing effect (Fernandes et al., 2018). When a user shares
  information worth remembering, encode it as two files: a structured fact file
  ($MEMORY_DIR/facts/{anchor}.md) and a visual scene file ($MEMORY_DIR/scenes/{anchor}.md).
  Use when a user shares personal details, preferences, events, or any information
  they may want recalled later. Includes a three-state retrieval protocol for
  answering memory questions with calibrated confidence.
---

# Drawing-Memory Skill: Letta Code / Context Repository Edition

## Architecture Note

This skill is designed for Letta Code agents with memfs enabled. Memory files
are written to the agent's git-backed memory directory using two environment
variables injected automatically at session start:

  $MEMORY_DIR  -- absolute path to ~/.letta/agents/<agent-id>/memory/
  $AGENT_ID    -- the agent's own ID

  $MEMORY_DIR/facts/{anchor}.md   -- structured paraphrase, always written
  $MEMORY_DIR/scenes/{anchor}.md  -- visual scene description, written for FULL path

Client-side tools note: Write, Edit, and Bash are client-side tools. They
execute only when the agent is accessed through the Letta Code TUI or LettaBot,
not through bare API calls. The full encoding workflow (Write files + git commit)
requires an interactive session.

This is distinct from the Letta V1 API agent version (archival_memory_insert).
File-based storage enables complete enumeration via Glob -- solving multi-session
aggregation without a symbolic index workaround.

## Research Foundation

Inspired by the drawing effect (Fernandes, Wammes, & Meade, 2018): drawing
produces stronger memory traces than writing because it creates integrated,
multi-representational traces. This skill translates that into dual-trace
file encoding -- a structured fact (verbal/semantic) paired with a vivid scene
(spatial/episodic).

  Citation: Fernandes, M. A., Wammes, J. D., & Meade, M. E. (2018).
  The surprisingly powerful influence of drawing on memory. Current
  Directions in Psychological Science, 27(5), 302-308.

The dual-trace principle was empirically validated on LongMemEval-S (Wang et
al., 2024) in the letta_v1_agent / archival memory condition (C6 vs C7,
paired analysis over 99 questions, bootstrap p < 0.0001). Results from that
evaluation:

  Overall accuracy:    +20.2pp  (73.7% vs 53.5%)
  Temporal reasoning:  +40pp    -- scenes anchor events in time
  Knowledge-update:    +25pp    -- scenes signal which state is current
  Multi-session:       +30pp    -- scenes bind scattered entries into threads
  Single-session:        0pp    -- null result (expected: scenes add nothing
                                   when one lookup suffices)
  Token cost:          -3.3%    -- higher cache hit rate offsets scene overhead

The null result on single-session is the mechanistic fingerprint: scenes
contribute specifically when memory must be aggregated, sequenced, or resolved
across multiple entries.

Note on scope: the above results are from the archival_memory_insert variant
(vector-DB storage). This file-based adaptation preserves the dual-trace
encoding structure and is expected to retain the same mechanistic advantage.
Controlled experimental validation of the file-based variant is future work.
Full results and docs: https://github.com/sternb12/agent_draw_skills

## Auto-Activation: System Prompt Directive (Required)

Skills are never auto-triggered by Letta Code. The agent sees a one-line
description of each skill but only loads it when it explicitly invokes the
Skill tool. To make the drawing-memory skill activate reliably on a semantic
condition (user shares personal information), you must add a directive to the
agent's system prompt.

Recommended approach: write the activation rule to
$MEMORY_DIR/system/memory-protocol.md at agent setup time. Files in system/
are pinned to the agent's context on every turn, making the directive visible
on every interaction without consuming skill load budget.

A ready-to-use template is in references/memory-protocol-template.md.
Copy its contents into $MEMORY_DIR/system/memory-protocol.md and commit.

Without this directive, the agent will default to conversational response
when a user shares personal information, rather than invoking the skill.

## When to Use This Skill

Activate when a user shares information they may want recalled later:
- Personal facts (age, job, family, location, health)
- Preferences (food, hobbies, habits, routines)
- Events (appointments, milestones, things that happened)
- Decisions and changes (switched jobs, moved, updated a plan)
- Anything the user explicitly asks you to remember

Do not use for: general conversation, world knowledge, transient details the
user will not need again, or information already stored -- check for duplicates
before encoding.

---

## ENCODING WORKFLOW

### Evidence Assessment

Score each piece of information before encoding:

  Category          | 0 pts        | 1 pt        | 2 pts       | 3 pts
  ------------------|--------------|-------------|-------------|------------------
  Source reliability| Unknown      | Inferred    | Stated      | Confirmed by user
  Specificity       | Vague        | Partial     | Specific    | Precise w/ ID
  Repetition        | First mention| Twice       | Multiple    | Core/recurring
  Consistency       | Contradicts  | No context  | Consistent  | Reinforces stored

Routing:
  0-4:  DROP -- do not encode
  5-7:  STREAMLINED -- fact file only, no scene
  8-12: FULL -- both fact and scene files

Stakes override: For high-stakes discrete items (codes, credentials, safety
procedures, medical info) with score >= 5, default to FULL regardless of
STREAMLINED routing.

Coverage calibration: When in doubt between DROP and STREAMLINED, prefer
STREAMLINED. When in doubt between STREAMLINED and FULL, prefer FULL for
anything with a temporal or relational dimension. Missed encodings cannot
be recovered; over-encoding is correctable.

Contradiction rule: If new information conflicts with an existing fact file,
do NOT overwrite silently. Surface the conflict to the user first.

### Step 1: Check for Duplicates

Before writing, check whether a fact file already exists for this anchor:

  Read $MEMORY_DIR/facts/{anchor}.md

If the file exists: compare content with new information. If consistent,
skip encoding or update the file with the new information. If contradictory,
surface the conflict to the user before proceeding.

If the file does not exist (Read returns an error): proceed to encoding.

### Step 2: Classify Information Type

  - Discrete    -- single items with clear identifiers (codes, names, dates)
  - Compositional -- multi-part concepts with several components
  - Relational  -- procedures, sequences, causal chains with dependencies

### Step 3: Write the Fact File

Write to: $MEMORY_DIR/facts/{anchor}.md  (use absolute path with Write tool)
Filename: lowercase, hyphens for spaces (e.g., work-occupation.md,
hobby-running.md, health-therapy.md). The fact file and scene file for
the same item share the same base name.

Frontmatter:
  ---
  description: [Searchable summary -- must contain the anchor term and any
    specific identifier that makes this file findable by topic]
  type: [discrete | compositional | relational]
  linked_scene: $MEMORY_DIR/scenes/{anchor}.md    # omit for STREAMLINED path
  confidence: [HIGH | MEDIUM | LOW]
  evidence_score: [0-12]
  stored: [YYYY-MM-DD]
  ---

Content:
  - Paraphrase the information in your own words -- do not copy verbatim
  - For compositional items: list all components explicitly before writing.
    Confirm every component appears in the paraphrase before saving.
  - For relational items: preserve sequence order and dependencies
  - The frontmatter description MUST contain the most specific identifier
    for this information (the anchor term that makes it findable)

### Step 4: Write the Scene File (FULL path only)

Write to: $MEMORY_DIR/scenes/{anchor}.md  (use absolute path with Write tool)
Same base name as the fact file.

Frontmatter:
  ---
  description: Scene cue for [anchor] -- [brief topic context]
  type: scene
  linked_fact: $MEMORY_DIR/facts/{anchor}.md
  confidence: [HIGH | MEDIUM | LOW]
  stored: [YYYY-MM-DD]
  ---

Content format:
  Picture: [One-sentence scene. A concrete object with a distinctive visual
  mark, in a specific setting. Embed the key information explicitly as a
  quote, label, or speech bubble within the scene.]

  Sketch steps: (1) Draw [object in setting], (2) Add [distinctive mark]
  on [location], (3) Embed "[key quote]" as [sign/label/speech bubble]

  (Mnemonic depiction only. Not evidence.)

Scene quality rules:
  - Use concrete, visually specific objects (not abstractions)
  - The distinctive mark must be unique to THIS scene
  - Key information must appear explicitly in the scene as text
  - Anchor term must appear somewhere in the scene

Temporal anchor rule (critical): When the information has any time dimension
-- a date, a sequence, a before/after relationship, a change over time -- the
scene MUST embed a concrete temporal cue. Examples:
  - "a rainy Sunday afternoon, the first session with this kit"
  - "the November planner open, weekly appointments circled"
  - "the calendar on the wall shows April, inbox already overflowing"
Without a temporal anchor, scenes fail at the questions they're most needed
for: what happened first, what changed, when did this start.

Contextual binding rule: For information likely to recur across sessions
(jobs, health, relationships, recurring topics), the scene should capture
WHY this came up -- the emotional weight, the setting, the context that
makes this instance recognizable as part of a larger thread.

Scene validation: Before writing, confirm the scene contains no new specific
facts (dates, names, numbers, places) not present in the user's original
information. Metaphorical objects are permitted. Invented specifics are not.

### Step 5: Commit

After writing both files, commit to the agent's memory repo.
$MEMORY_DIR is injected automatically -- no need to compute the agent ID.

  git -C "$MEMORY_DIR" add facts/ scenes/
  git -C "$MEMORY_DIR" commit -m "mem: store {anchor} ({type}, evidence:{score})"
  git -C "$MEMORY_DIR" push

Commit message formats:
  mem: store {anchor} ({type}, evidence:{score})
  mem: store {anchor} ({type}, evidence:{score}, stakes override)
  mem: store {anchor} ({type}, evidence:{score}, streamlined)
  mem: update {anchor} (user-confirmed correction, evidence:{score})
  mem: upgrade {anchor} (added scene, evidence:{score})

### STREAMLINED Path (Evidence Score 5-7)

Write the fact file with full frontmatter. Omit the linked_scene field
entirely. Skip the scene file. These items retrieve at State B (medium
confidence). If the user later confirms or repeats the information, re-score
and consider upgrading to FULL by adding a scene file.

---

## RETRIEVAL PROTOCOL (Three-State)

When a user asks a question that may be answerable from stored memory,
follow this protocol before responding.

### Step 1: Identify the Anchor

Determine the most specific anchor term for what is being asked.
Examples: "work-occupation", "hobby-running", "health-therapy-dr-smith",
"personal-age", "relationship-partner".

For aggregate questions ("how many times", "most often", "has the user
ever", "what changed"), identify the domain anchor and proceed to
multi-session scan (Step 3).

### Step 2: Check for Fact File

  Read $MEMORY_DIR/facts/{anchor}.md

Three outcomes:

STATE A -- Fact file found AND scene file exists:
  1. Read $MEMORY_DIR/scenes/{anchor}.md
  2. Mentally reconstruct the scene before formulating your answer
  3. Answer with high confidence, drawing on both traces
  4. You may briefly reference the scene context if it aids the answer
     (e.g., "I recall this from a specific moment -- [scene detail]")

STATE B -- Fact file found but NO scene file:
  1. Answer from the fact file content
  2. Answer with medium confidence (note this is a STREAMLINED entry)
  3. Do not fabricate a scene

STATE C -- No fact file found for this anchor:
  1. Do not guess or confabulate
  2. Respond: "I don't have that stored in my memory."
  3. Optionally ask if the user would like to tell you so you can
     remember it going forward

### Step 3: Multi-Session Aggregation (for aggregate questions)

For questions requiring aggregation across multiple entries:

  1. Run: Glob(pattern="*.md", path="$MEMORY_DIR/facts/")
     Note: always use path= with $MEMORY_DIR, NOT a relative pattern like
     "facts/*.md" -- the working directory is not the memfs root.
  2. Read each file whose frontmatter description matches the topic
  3. Collect ALL matching entries (not just the first)
  4. For each matching fact file, check whether a corresponding scene
     file exists and read it -- scene temporal anchors are critical for
     sequencing entries chronologically
  5. Synthesize across all entries before answering

This enumeration is the key advantage of file-based storage over vector
search: complete recall, not probabilistic top-k retrieval.

Example aggregate questions that require this step:
  - "How many times have I mentioned X?"
  - "What changed about my job over time?"
  - "Which of my hobbies did I start first?"
  - "Have I ever talked about [topic]?"

### Step 4: Confidence Calibration

  High confidence (State A):  Both traces found, scene reconstructed
  Medium confidence (State B): Fact only, no scene
  Abstain (State C):           No file found -- do not guess

When in doubt, abstain. A calibrated "I don't have that stored" is more
useful than a confabulated answer.

---

## WORKED EXAMPLES (Real LME-S Evaluation Results)

See references/worked-examples.md for three annotated examples from a
controlled evaluation showing each retrieval mechanism in action:
  1. Multi-session aggregation (scene as contextual binding cue)
  2. Knowledge-update / change tracking (scene signals current state)
  3. Temporal reasoning (scene enables self-correction via date anchor)

---

## QUICK REFERENCE

ENCODING DECISION FLOW:

  User shares information
        |
        v
  Check $MEMORY_DIR/facts/{anchor}.md -- exists? -- compare & resolve conflict
        |
        v (no duplicate)
  Evidence Assessment (0-12)
        |
   +----+------------+
   v    v            v
  0-4  5-7          8-12
  DROP STREAMLINED  FULL
       |            |
       v            v
     Fact         Fact + Scene
     file         files
       |            |
       v            v
     Commit       Commit

  * Stakes override: discrete + identifier + consequences -> FULL at score >= 5
  * Contradiction: pause, surface to user, do not encode silently
  * Coverage: prefer STREAMLINED over DROP, FULL over STREAMLINED when temporal

RETRIEVAL DECISION FLOW:

  User asks memory question
        |
        v
  Identify anchor term
        |
        v
  Read $MEMORY_DIR/facts/{anchor}.md
        |
   +----+----+
   v         v
  Found    Not found
   |           |
   v           v
  Check      STATE C: abstain
  $MEMORY_DIR/scenes/{anchor}.md
   |
  +----+
  v         v
 Found    Not found
  |           |
  v           v
STATE A    STATE B
(reconstruct  (fact only,
 scene, high)  medium)

  For aggregate questions: Glob(pattern="*.md", path="$MEMORY_DIR/facts/"),
  read all matching files, use scene temporal anchors to sequence,
  synthesize before answering. Never use relative Glob patterns.

FILE STRUCTURE (in agent memfs):

  $MEMORY_DIR/                     (= ~/.letta/agents/<id>/memory/)
  +-- system/
  |   +-- persona.md               <- always in context (pinned)
  |   +-- human.md                 <- always in context (pinned)
  |   +-- memory-protocol.md       <- always in context (auto-activation rule)
  +-- facts/
  |   +-- {anchor}.md              <- paraphrase + frontmatter
  +-- scenes/
      +-- {anchor}.md              <- scene + sketch steps + frontmatter

FRONTMATTER (fact file):
  ---
  description: [summary with ANCHOR TERM and specific identifier]
  type: [discrete | compositional | relational]
  linked_scene: $MEMORY_DIR/scenes/{anchor}.md    # omit for streamlined
  confidence: [HIGH | MEDIUM | LOW]
  evidence_score: [0-12]
  stored: [YYYY-MM-DD]
  ---

FRONTMATTER (scene file):
  ---
  description: Scene cue for [ANCHOR] -- [topic]
  type: scene
  linked_fact: $MEMORY_DIR/facts/{anchor}.md
  confidence: [HIGH | MEDIUM | LOW]
  stored: [YYYY-MM-DD]
  ---

SCENE FORMAT:
  Picture: [concrete object + distinctive mark + embedded key info +
            TEMPORAL CUE if information has time dimension]
  Sketch steps: (1) Draw [object], (2) Add [mark] on [location],
  (3) Embed "[quote]" as [sign/label/speech bubble]
  (Mnemonic depiction only. Not evidence.)

COMMIT FORMAT:
  mem: store {anchor} ({type}, evidence:{score})
  mem: update {anchor} (user-confirmed correction, evidence:{score})
  mem: upgrade {anchor} (added scene, evidence:{score})
