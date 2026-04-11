---
name: drawing-memory
description: >
  Scene layer for Letta Code agents. Adds dedicated fact records (RECORD
  path) or paired fact-plus-scene files (FULL path) to Letta Code's
  existing memfs, depending on evidence score. When a design decision is
  made, an incident is investigated, a convention is established with
  rationale, a pattern is observed, a learning progression becomes
  visible, or a preference is stated with reasoning, this skill routes
  the event through an evidence-scoring step and writes the appropriate
  trace. Extends (does not replace) /init, /remember, and the memory
  check reminder. Includes three-state retrieval protocol for answering
  questions from stored coding memory with calibrated confidence.
---

# Drawing-Memory Skill (Coding Adaptation)

## Research foundation

Adapted from the dual-trace memory encoding protocol described in
"Drawing on Memory: Dual-Trace Encoding Improves Cross-Session Recall in
LLM Agents." The mechanism is analogous to elaborative encoding from the
cognitive-psychology literature: generating a concrete narrative
reconstruction of the moment a decision was made or an insight occurred
produces a richer, more distinctive memory trace than the fact alone,
and the scene functions as a contextual binding cue at retrieval time
(Tulving & Thomson, 1973; Fernandes, Wammes, & Meade, 2018). The parent
protocol was evaluated on LongMemEval-S at 73.7% vs 53.5% overall
accuracy against a fact-only control (Stern et al., 2026, §4). The
coding adaptation in this skill is pilot-validated only; see §5 of the
manuscript and `PILOT_TESTS.md` in this directory for the four manual
tests that motivated the current design.

## Architecture note

This skill is a **scene layer on top of Letta Code's existing memfs**,
not a standalone memory system. It extends the memory infrastructure
that `/init`, `/remember`, and the memory check reminder already
provide.

**What Letta Code already handles (this skill does NOT replace):**
- `system/human.md` -- developer preferences, profile
- `system/{project}/overview.md` -- project overview, tech stack
- `system/{project}/commands.md` -- build, test, lint workflows
- `system/{project}/conventions.md` -- basic project conventions
- Memory check reminder -- periodic silent fact updates
- `/remember` -- explicit memory storage
- `/init` -- project bootstrapping

**What this skill adds:**
- A `scenes/` directory with narrative scene traces
- Dedicated fact files for decisions, incidents, patterns, and learning
  progressions that do not fit in existing Letta Code blocks
- Evidence scoring to determine encoding depth
- A three-state retrieval protocol with confidence calibration

**Tools required:** Write, Read, Glob, Bash (for git), Memory tool (for
updating existing Letta Code blocks).

---

## ENCODING WORKFLOW

### When to encode

This skill activates for information with rationale depth, temporal
dimension, or cross-session significance -- cases where a scene trace
is expected to improve later retrieval. Routine facts are handled by
Letta Code's existing memory mechanisms and should not trigger this
skill.

  Activate when you observe:
  - A design decision with stated rationale
  - A non-trivial bug investigated and resolved (or root cause identified)
  - A convention established with an origin story or significant rationale
  - A library, framework, or tool choice with reasoning
  - A tradeoff explicitly discussed
  - A codebase-level pattern (positive or problematic)
  - A recurring developer pattern on its second or later occurrence
  - A visible skill progression (developer learned something new)
  - A developer correcting a repeated mistake

  Do NOT activate (let Letta Code handle normally):
  - Routine edits (renames, formatting, simple refactors)
  - Implementation details with no rationale
  - Transient debugging (print statements, console.logs)
  - Simple preferences without rationale (human.md handles these)
  - Basic conventions without origin stories (conventions.md handles these)
  - General programming knowledge
  - Information already stored

### Step 1: Check existing memory

Before writing anything, check whether Letta Code already has this
stored:

  Read system/{project}/conventions.md   -- for conventions
  Read system/human.md                   -- for preferences
  Read decisions/{anchor}.md             -- for decisions
  Read incidents/{anchor}.md             -- for incidents
  Read patterns/{anchor}.md              -- for patterns
  Read learning/{anchor}.md              -- for learning progressions

If the fact already exists in a Letta Code block, do not duplicate it.
If routing FULL, add a scene that links back to the existing location.
If the fact contradicts existing memory, surface the conflict to the
developer. Do NOT overwrite silently.

If the fact exists in a dedicated file from an earlier session, update
that file in place rather than writing a second one. Confidence
upgrades (MEDIUM -> HIGH), status changes, and new timeline entries all
belong on the existing record.

### Step 2: Score evidence (0-12)

The score determines encoding depth, not whether to store at all. Letta
Code's memory check reminder handles baseline fact storage. This score
determines whether this skill adds a scene.

  Durability (0-3):
    0 = transient (will change in hours)
    1 = short-lived (days, current sprint)
    2 = medium-term (quarter, current release cycle)
    3 = long-term (architectural, persists across versions)

  Scope (0-3):
    0 = single line or file
    1 = single module or component
    2 = cross-module or cross-service
    3 = system-wide or cross-project

  Rationale-richness (0-3):
    0 = no "why" -- just what was done
    1 = implicit rationale (inferable from context)
    2 = explicit rationale stated
    3 = rationale + alternatives considered + tradeoffs discussed

  Retrieval likelihood (0-3):
    0 = unlikely to be asked about
    1 = might come up
    2 = will likely be needed again
    3 = critical -- will be asked repeatedly

  Total = sum of all four (0-12)

### Step 3: Route

  0-4:   SKIP    -- let Letta Code's normal memory handle it. Do not
                    invoke this skill's encoding workflow. If the item
                    is worth retaining, the agent's default behavior
                    (updating system/human.md or system/{project}/
                    conventions.md via the Memory tool) handles it.
  5-7:   RECORD  -- write a dedicated fact file if one does not already
                    exist in a Letta Code block. No scene.
  8-12:  FULL    -- write a dedicated fact file (if needed) AND a scene
                    file.

  Stakes override: For incidents (outages, data loss, security issues)
  or irreversible decisions (database migrations, API contracts,
  framework switches), route FULL once the score reaches 5. The cost of
  an extra scene file is small; the cost of abstaining on a security
  incident is not.

  Coverage calibration: Prefer RECORD over SKIP. Prefer FULL over
  RECORD for anything with a temporal or cross-session dimension. The
  underlying fact may be captured by Letta Code, but the scene context
  will be lost if not encoded now.

### Step 4: Determine where the fact lives

The fact may already have a home in a Letta Code block, or it may need
a dedicated file. Check this before writing:

  | Information            | Fact location                              | Scene location                |
  |---|---|---|
  | Simple convention      | system/{project}/conventions.md (existing) | scenes/conventions/{anchor}.md |
  | Convention with origin | system/{project}/conventions.md (existing) | scenes/conventions/{anchor}.md |
  | Simple preference      | system/human.md (existing)                 | No scene (RECORD)             |
  | Preference w/ rationale| system/human.md (existing)                 | scenes/preferences/{anchor}.md|
  | Design decision        | decisions/{anchor}.md (new)                | scenes/decisions/{anchor}.md  |
  | Incident               | incidents/{anchor}.md (new)                | scenes/incidents/{anchor}.md  |
  | Recurring pattern      | patterns/{anchor}.md (new)                 | scenes/patterns/{anchor}.md   |
  | Learning progression   | learning/{anchor}.md (new)                 | scenes/learning/{anchor}.md   |

Two asymmetries to notice. First, **conventions and preferences are
special**: Letta Code already has a natural home for them in the
`system/` block, so this skill writes only the scene and links back to
the existing block -- it never duplicates the fact. Second, **anchors
drop the type prefix**: use `structured-logging-json`, not
`decision-structured-logging-json`, because the containing folder
already carries the type.

*Anchor naming for incidents.* Production incidents with a known date
take a date prefix so they sort chronologically in the directory
listing (`incidents/2026-03-15-auth-timeout.md`). In-session incidents
without an external date stay un-prefixed
(`incidents/auth-middleware-401-race.md`). The same date-prefix
convention applies to the matching scene file when one is written.

If the fact belongs in an existing Letta Code block, update it using
the Memory tool (`str_replace` or `insert`). Do not duplicate.

If the fact needs a dedicated file, write it using the Write tool.

### Step 5: Write fact file (if needed)

Skip this step if the fact already exists in a Letta Code block. Only
write a dedicated fact file for information types that do not fit in
existing blocks: decisions, incidents, patterns, and learning
progressions.

Write to: `$MEMORY_DIR/{category}/{anchor}.md`

  ---
  description: [Searchable summary with anchor term and key identifier]
  type: [decision | incident | convention | preference | pattern | learning]
  linked_scene: scenes/{category}/{anchor}.md
  related: []
  confidence: [HIGH | MEDIUM | LOW]
  evidence_score: [0-12]
  stored: [YYYY-MM-DD]
  ---

  [Third-person paraphrase preserving all technical specifics.]

Content rules by type:
- **Decision**: include alternatives considered, rationale, tradeoffs.
- **Incident**: include symptoms, root cause (or current hypothesis),
  resolution or status, and investigation path.
- **Pattern**: include occurrences observed and trajectory.
- **Learning**: include before-state, after-state, and progression
  timeline.

For the RECORD path: omit the `linked_scene` field entirely. Everything
else is the same as FULL -- the RECORD format is a structural subset of
FULL, which makes upgrading a RECORD entry to FULL a matter of adding a
scene file and the one header line.

### Step 6: Write scene file (FULL path only)

Write to: `$MEMORY_DIR/scenes/{category}/{anchor}.md`

  ---
  description: Scene cue for {anchor} -- {brief topic}
  type: scene
  linked_fact: {path to fact -- either dedicated file or Letta Code block}
  confidence: [HIGH | MEDIUM | LOW]
  stored: [YYYY-MM-DD]
  ---

  [MOMENT:{anchor}]

  Moment: {One-paragraph narrative reconstruction. Include WHO was
  involved, WHAT was being worked on, WHERE in the codebase (specific
  file paths, function names, or module names from the conversation),
  WHEN relative to other events (commits, PRs, sprints, incidents),
  and WHY a decision was made or an insight occurred. Must contain at
  least one concrete code-level artifact mentioned in the original
  conversation.}

  Timeline: {anchor} -> {what happened} [{temporal marker}]
  Prior: {what came before, if relevant}
  After: {what followed or is expected to follow, if known}

  (Mnemonic reconstruction only. Not a code specification.)

The `linked_fact` field points to wherever the fact actually lives --
whether that is a dedicated file (`decisions/structured-logging-
json.md`) or an existing Letta Code block
(`system/{project}/conventions.md`).

**Scene quality rules.**

  *Temporal anchor rule.* When the information has any time dimension,
  the Timeline field MUST include a concrete temporal marker: a commit
  hash or PR number ("after PR #247"), a sprint or release boundary
  ("during v2.3 release"), relative sequencing ("before the API
  redesign"), an incident reference ("after the March 15 outage"), or
  a session reference ("in yesterday's session").

  *Codebase anchor rule.* Every scene MUST reference at least one
  concrete code-level artifact from the original conversation: a file
  path, function name, class name, error message, or module name.

  *Contextual binding rule.* For recurring topics, capture WHY this
  came up now -- what problem, symptom, or conversation triggered it.

  *Scene validation.* No invented specifics. Do not include file
  paths, function names, error messages, or technical details not
  present in the original conversation. Narrative framing is
  permitted. Fabricated code artifacts are not. The
  "(Mnemonic reconstruction only. Not a code specification.)" footer
  is there precisely so the scene is never mistaken for an authoritative
  source -- it exists to support recall, not to define behavior.

### Step 7: Commit

  git -C "$MEMORY_DIR" add decisions/ incidents/ patterns/ learning/ scenes/
  git -C "$MEMORY_DIR" commit -m "mem: store {category}/{anchor} ({type}, evidence:{score})"
  git -C "$MEMORY_DIR" push

Stage only the dual-trace directories. Do NOT use `git add .` -- it
would capture unrelated dirty state in `$MEMORY_DIR` (Memory-tool
updates to `system/` blocks, scratch files, partial work from other
tools). Listing the directories explicitly keeps each commit focused
on a single encoding event and makes the git history readable.

Use `mem: update` instead of `mem: store` when modifying an existing
file; use `mem: upgrade` when promoting a RECORD entry to FULL by
adding a scene.

After encoding, continue with the coding task. Do not repeat the
stored content back to the developer unless asked.

---

## RETRIEVAL PROTOCOL (three-state)

When the developer asks a question that may be answerable from stored
coding memory, follow this protocol before responding.

### Step 1: Identify anchor and search locations

Determine the most specific anchor term and where to search. Remember
to check BOTH Letta Code's existing blocks AND dual-trace files:

  "Why do we use X?"           -> decisions/ AND system/{project}/
  "Have we seen this before?"  -> incidents/
  "What's the convention?"     -> system/{project}/conventions.md AND
                                  convention scenes if they exist
  "How should I write tests?"  -> conventions.md + human.md + any
                                  preference scenes
  "What mistakes do I make?"   -> patterns/
  "What have I learned?"       -> learning/

### Step 2: Check for scene

After finding the fact (in a Letta Code block or a dedicated file),
check whether a scene exists:

  Glob(pattern="{anchor}*", path="$MEMORY_DIR/scenes/{likely-category}/")

If the category is uncertain, check the most likely directory first
based on the question type; if nothing is found, broaden to other
category directories before falling back to STATE C. A premature
abstain is just as bad as a confabulation.

Three outcomes:

  **STATE A -- Fact found AND scene exists:**
    1. Read the scene file.
    2. Reconstruct the narrative moment before formulating your answer.
    3. Answer with high confidence, drawing on both fact and scene.
    4. Reference the timeline or context from the scene when it aids
       the answer.

  **STATE B -- Fact found but NO scene:**
    1. Answer from the fact content.
    2. Answer with medium confidence.
    3. Do not fabricate a scene.

  **STATE C -- No fact found anywhere:**
    1. Do not guess or confabulate.
    2. Respond: "I don't have that stored in my memory."
    3. Optionally offer to encode it if the developer provides the
       information.

When answering, label your own retrieval state inline -- for example,
"(STATE A retrieval: fact + scene, high confidence)" or "(STATE C
retrieval: no fact found, abstaining)". This makes the confidence
assessment auditable and keeps calibration honest.

### Step 3: Multi-entry aggregation

For questions requiring aggregation across entries:

  1. Glob the relevant category directory:
       Glob(pattern="*.md", path="$MEMORY_DIR/{category}/")
     AND check relevant Letta Code blocks (conventions.md, etc.).
  2. Read each matching file.
  3. For each fact with a linked scene, read the scene -- use Timeline
     fields to sequence entries chronologically.
  4. Collect ALL matching entries before answering.
  5. Synthesize across entries, presenting temporal progression when
     relevant.

### Step 4: Confidence calibration

  High confidence (State A):   Fact + scene, narrative reconstructed
  Medium confidence (State B): Fact only, no scene context
  Abstain (State C):           No fact found -- do not guess

When in doubt, abstain. A calibrated "I don't have that stored" is
more useful than a confabulated answer.

For stale information: if a fact or scene references code artifacts
(file paths, function names) that may have changed, note the staleness
explicitly -- "I have this stored from [date], but the referenced
files may have changed since then."

---

## QUICK REFERENCE

EVIDENCE SCORING (0-12):
  Durability (0-3):          0=transient  1=sprint    2=quarter       3=architectural
  Scope (0-3):               0=line/file  1=module    2=cross-module  3=system-wide
  Rationale-richness (0-3):  0=none       1=implicit  2=explicit      3=full+tradeoffs
  Retrieval likelihood(0-3): 0=unlikely   1=possible  2=likely        3=critical

ROUTING:
  0-4:  SKIP   (let Letta Code handle)
  5-7:  RECORD (fact only, no scene)
  8-12: FULL   (fact + scene)
  Stakes override: incidents + irreversible decisions -> FULL at >= 5

WHERE FACTS LIVE:
  Simple conventions     -> system/{project}/conventions.md (existing)
  Simple preferences     -> system/human.md (existing)
  Decisions              -> decisions/{anchor}.md (new file)
  Incidents              -> incidents/{anchor}.md (new file)
                            (date-prefix production incidents:
                             incidents/2026-03-15-auth-timeout.md)
  Patterns               -> patterns/{anchor}.md (new file)
  Learning progressions  -> learning/{anchor}.md (new file)

WHERE SCENES LIVE:
  $MEMORY_DIR/scenes/{category}/{anchor}.md
  Categories: decisions/, incidents/, conventions/, preferences/,
              patterns/, learning/

SCENE FORMAT:
  [MOMENT:{anchor}]
  Moment:   {WHO + WHAT + WHERE in codebase + WHEN + WHY}
  Timeline: {anchor} -> {event} [{temporal marker}]
  Prior:    {context before}
  After:    {what followed}
  (Mnemonic reconstruction only. Not a code specification.)

RETRIEVAL:
  1. Find the fact (Letta Code block or dedicated file)
  2. Check scenes/{category}/ for a matching scene
     (broaden to other categories if uncertain before STATE C)
  STATE A: fact + scene    -> reconstruct moment, high confidence
  STATE B: fact only       -> medium confidence
  STATE C: nothing         -> abstain, do not guess
  Aggregate: Glob category + check Letta Code blocks, read all,
             sequence via Timeline, synthesize.

GIT STAGING (Step 7):
  git -C "$MEMORY_DIR" add decisions/ incidents/ patterns/ learning/ scenes/
  Do NOT use `git add .` -- it captures unrelated dirty state.

COMMIT FORMAT:
  mem: store {category}/{anchor} ({type}, evidence:{score})
  mem: update {category}/{anchor} ({reason}, evidence:{score})
  mem: upgrade {category}/{anchor} (added scene, evidence:{score})

---

## Known limitation: auto-activation

Pilot testing (2026-03-24; see `PILOT_TESTS.md` in this directory)
identified a reproducible failure mode in how Letta Code loads this
skill on a cold start. When a session begins, the agent's default
memory behavior (update `human.md`) fires before the skill-loading step
is ever triggered, even with a directive in `system/` explicitly
pointing at the skill. The consequence is that architectural decisions
presented in the first message of a session are routed to `human.md`
rather than to `decisions/`, and no scene is written.

The validated workaround, used in every passing pilot test, is a
single explicit invocation at the start of the session:

  "Please load your drawing-memory skill."

After that one invocation the skill fires automatically on every
scene-worthy input for the remainder of the session. This is the only
validated setup path for v1 of the skill.

A cleaner fix is in progress. The v2 design moves encoding out of the
conversation entirely and runs it as a post-session sub-agent fired
from a Stop hook -- a pattern borrowed from Claude Code's autoDream.
That design eliminates the skill-loading indirection by not requiring
the skill to fire mid-conversation at all. See
`references/coding_dual_trace_v2.md` (in this skill's references
directory) for the full sketch; v2 is planned but not yet tested.

---

## References

Fernandes, M. A., Wammes, J. D., & Meade, M. E. (2018). The surprisingly
powerful influence of drawing on memory. *Current Directions in
Psychological Science*, 27(5), 302-308.

Tulving, E., & Thomson, D. M. (1973). Encoding specificity and
retrieval processes in episodic memory. *Psychological Review*, 80(5),
352-373.

Stern, B., et al. (2026). Drawing on memory: Dual-trace encoding
improves cross-session recall in LLM agents. Manuscript. See
`https://github.com/sternb12/agent_draw_skills` for the parent protocol
and its evaluation on LongMemEval-S.
