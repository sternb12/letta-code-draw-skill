# Coding Dual-Trace Memory: Architecture Plan

Adaptation of dual-trace memory encoding for coding tasks on Letta Code agents.
Designed to sit on top of Letta Code's existing memfs infrastructure, adding
a scene layer to the memory system that already handles facts, conventions,
and developer preferences.

**Target agent:** Letta Code (git-backed memfs, Memory tool, /init, /remember)

**Scope:** Both project-level memory and developer-level memory, extending
rather than replacing what Letta Code already provides.

---

## 1. What Letta Code Already Provides

Letta Code has a mature, git-backed memory filesystem (memfs) at
`~/.letta/agents/{agentId}/memory/`. Before designing the coding dual-trace
system, we need to be clear about what already exists so we don't
duplicate functionality.

### Already built (do not rebuild)

| Capability | How it works | Where it lives |
|---|---|---|
| Git-backed persistence | All memory is markdown + YAML frontmatter, version-controlled with auto-push | memfs root |
| Project bootstrapping | `/init` reads git history, package files, README, creates skeleton | system/{project}/ |
| Project conventions | `/init` creates conventions.md from docs and codebase | system/{project}/conventions.md |
| Project overview | `/init` creates overview.md with tech stack, structure | system/{project}/overview.md |
| Project commands | `/init` creates commands.md with build/test/lint workflows | system/{project}/commands.md |
| Developer preferences | human.md stores name, role, preferences, working style | system/human.md |
| Agent personality | persona.md evolves over time as agent learns developer | system/persona.md |
| Explicit memory | `/remember` stores information via Memory tool (str_replace, insert, create) | any block |
| Automatic learning | Periodic memory check reminder prompts agent to silently update blocks | during conversation |
| Memory tree view | memory_filesystem block renders current tree in context | read-only block |
| Commit audit trail | All changes committed with agent attribution | git log |
| Pinned system files | Files in system/ are always in the agent's context | system/ |

### What's missing (what this skill adds)

Letta Code stores facts. It stores conventions. It stores preferences. What
it does NOT do is:

1. **Generate a paired scene trace** that anchors facts to their moment and
   context, enabling temporal reasoning and cross-session aggregation
2. **Score evidence** to decide how deeply to encode (STREAMLINED vs. FULL)
3. **Use a retrieval protocol** with calibrated confidence states (A/B/C)
4. **Reconstruct context at retrieval time** before answering
5. **Track debugging incidents** as structured narratives with investigation
   timelines
6. **Track developer learning progressions** over time

The dual-trace system is a scene layer on top of Letta Code's fact storage,
not a replacement for it.

---

## 2. Integration Architecture

### Principle: extend, don't replace

The coding dual-trace skill does NOT create its own facts/ directory or its
own developer/ directory. It uses Letta Code's existing memory structure and
adds a `scenes/` directory alongside it. Facts continue to be stored in
the places Letta Code already puts them. Scenes are the new addition.

### Directory structure (additions in bold)

```
$MEMORY_DIR/
+-- system/                              <- Letta Code (existing, pinned)
|   +-- persona.md                       <- existing
|   +-- human.md                         <- existing (developer prefs live here)
|   +-- {project}/                       <- existing (created by /init)
|   |   +-- overview.md                  <- existing
|   |   +-- commands.md                  <- existing
|   |   +-- conventions.md              <- existing
|   +-- coding-memory-protocol.md        <- NEW (auto-activation directive)
|
+-- scenes/                              <- NEW (all scene files live here)
|   +-- decision/{anchor}.md             <- NEW (decision moment reconstructions)
|   +-- debug/{anchor}.md                <- NEW (debugging incident narratives)
|   +-- convention/{anchor}.md           <- NEW (convention origin stories)
|   +-- learning/{anchor}.md             <- NEW (developer skill progressions)
|   +-- pattern/{anchor}.md              <- NEW (recurring behavior observations)
|
+-- debug/                               <- NEW (structured incident files)
|   +-- {anchor}.md                      <- fact files for debugging incidents
|
+-- decision/                            <- NEW (structured decision records)
|   +-- {anchor}.md                      <- fact files for design decisions
|
+-- learning/                            <- NEW (developer learning tracks)
|   +-- {anchor}.md                      <- fact files for skill progressions
|
+-- pattern/                             <- NEW (recurring patterns)
    +-- {anchor}.md                      <- fact files for observed patterns
```

### What goes where

| Information type | Fact storage | Scene storage |
|---|---|---|
| Project overview, tech stack | system/{project}/overview.md (existing, via /init) | No scene needed |
| Build/test/lint commands | system/{project}/commands.md (existing, via /init) | No scene needed |
| Simple conventions | system/{project}/conventions.md (existing, via /init or /remember) | No scene needed |
| Convention with origin story | system/{project}/conventions.md (existing) + dedicated fact if complex | scenes/convention/{anchor}.md |
| Design decision with rationale | decision/{anchor}.md | scenes/decision/{anchor}.md |
| Debugging incident | debug/{anchor}.md | scenes/debug/{anchor}.md |
| Developer preference | system/human.md (existing, via /remember) | No scene needed (STREAMLINED) |
| Developer preference with rationale | system/human.md (existing) | scenes/preference/{anchor}.md (if FULL) |
| Recurring pattern | pattern/{anchor}.md | scenes/pattern/{anchor}.md |
| Learning progression | learning/{anchor}.md | scenes/learning/{anchor}.md |

### Key integration rules

**Rule 1: Don't duplicate what /init and /remember already handle.**
Simple facts (project overview, basic conventions, developer name/role)
belong in Letta Code's existing blocks. The dual-trace skill activates
for information with rationale depth, temporal dimension, or cross-session
significance -- the cases where scene traces provide retrieval value.

**Rule 2: Scenes always link back to their fact source.**
Every scene file includes a `linked_fact` field pointing to where the
fact lives -- whether that's a dedicated fact file (decision/foo.md) or
an existing Letta Code block (system/{project}/conventions.md). This
linking is what makes the three-state retrieval protocol work.

**Rule 3: Use the Memory tool for existing blocks, Write for new files.**
When updating system/human.md or system/{project}/conventions.md, use
the Memory tool (str_replace, insert) that Letta Code already provides.
When creating new scene files or dedicated fact files, use the Write tool.
Both are committed via git.

**Rule 4: The auto-activation directive coexists with the memory check
reminder.** Letta Code already injects periodic memory check reminders
that prompt the agent to silently update blocks. The coding-memory-protocol.md
directive adds scene-specific triggers on top of this. They don't conflict:
the memory check reminder handles routine fact updates; the dual-trace
skill handles deeper encoding when scene-worthy events are detected.

---

## 3. What the Skill Actually Does (Scope)

Given what Letta Code already provides, the coding dual-trace skill has a
focused scope:

### The skill's job

1. **Detect scene-worthy events** during coding sessions (decisions with
   rationale, debugging incidents, learning progressions, recurring patterns)
2. **Score evidence** to decide STREAMLINED (fact only) vs. FULL (fact + scene)
3. **Write scene files** in the scenes/ directory with narrative Moment format
4. **Write dedicated fact files** for information types that don't fit
   naturally into Letta Code's existing blocks (incidents, decisions with
   complex rationale, learning progressions)
5. **Follow the three-state retrieval protocol** when answering questions,
   using scene reconstruction for confidence calibration

### NOT the skill's job (Letta Code handles these)

- Bootstrapping project knowledge (/init does this)
- Storing simple preferences (/remember and memory check do this)
- Storing basic conventions (/init creates conventions.md)
- Managing the git commit/push cycle (Memory tool handles this)
- Rendering the memory tree (memory_filesystem block does this)
- Storing developer name, role, basic profile (human.md does this)

---

## 4. Evidence Scoring

The scoring system determines whether an event is worth creating a scene
for. It does NOT determine whether to store the fact at all -- Letta Code's
memory check reminder and /remember handle routine fact storage. The scoring
determines the **encoding depth**: fact only (STREAMLINED) or fact + scene
(FULL).

### Scoring Dimensions (0-12)

| Dimension | 0 | 1 | 2 | 3 |
|---|---|---|---|---|
| **Durability** | Transient (will change in hours) | Short-lived (days/sprint) | Medium-term (quarter) | Long-term (architectural, persists across versions) |
| **Scope** | Single line/file | Single module/component | Cross-module/service | System-wide / cross-project |
| **Rationale-richness** | No "why" -- just what was done | Implicit rationale | Explicit rationale stated | Rationale + alternatives considered + tradeoffs discussed |
| **Retrieval likelihood** | Unlikely to be asked about | Might come up | Will likely be needed | Critical -- will be asked repeatedly |

**Total = Durability + Scope + Rationale-richness + Retrieval likelihood (0-12)**

### Routing

```
0-4:   SKIP       -- let Letta Code's normal memory handle it
5-7:   STREAMLINED -- write a dedicated fact file if one doesn't exist,
                      but no scene. Update existing blocks if appropriate.
8-12:  FULL       -- write dedicated fact file + scene file
```

Note the change from DROP to SKIP: at 0-4, the information isn't dropped --
it may still be captured by Letta Code's memory check reminder or /remember.
The dual-trace skill simply doesn't add to the encoding.

### Stakes override

For incidents (outages, data loss, security issues) or decisions that are
expensive to reverse (database migrations, API contract changes, framework
switches): if score >= 5, route FULL regardless of STREAMLINED.

### Coverage calibration

Prefer STREAMLINED over SKIP, FULL over STREAMLINED for anything with a
temporal or cross-session dimension. Missed scene encodings of architectural
rationale cannot be recovered after the conversation ends. The underlying
fact may still be captured by Letta Code, but the scene context will be lost.

---

## 5. Information Types

### Decision

A choice between alternatives with stated or inferable rationale.

Encoding: Write to decision/{anchor}.md. Always capture alternatives
considered, rationale, and tradeoffs. Almost always FULL -- the rationale
is the most valuable part and scenes anchor the decision to its moment.

### Incident

A debugging session, outage, failure, or surprising behavior.

Encoding: Write to debug/{anchor}.md. Capture symptoms, root cause,
resolution, and investigation path. Strongest candidates for FULL --
debugging is inherently sequential and temporal.

### Convention (with origin story)

A project convention that has significant rationale behind it.

Encoding: Simple conventions stay in system/{project}/conventions.md
(Letta Code's existing location). A convention with a story ("We adopted
this after the incident where...") gets a scene in scenes/convention/
that links back to conventions.md. The fact stays where Letta Code put it;
the scene adds the context.

### Pattern

A recurring behavior, approach, or tendency observed across sessions.

Encoding: Write to pattern/{anchor}.md. On first observation, STREAMLINED.
On second observation, upgrade to FULL. The scene captures the moment of
recognition.

### Learning

A developer skill transition observed over time.

Encoding: Write to learning/{anchor}.md. Always FULL -- the temporal
dimension is the whole point. Scene anchors the progression to specific
sessions and moments.

### Preference (with rationale)

Most preferences are handled by Letta Code's human.md. When a developer
explains WHY they prefer something ("I always want explicit error handling
because I've been burned by swallowed exceptions in production"), the
dual-trace skill adds a scene in scenes/preference/ that links back to
human.md. The preference stays in human.md; the scene adds the rationale
context.

---

## 6. Scene Vocabulary for Code

### The shift: from visual metaphors to narrative reconstruction

Instead of "Picture: a padlock with '443' on it," coding scenes use
**narrative reconstructions of moments** -- the debugging session, the
architecture discussion, the moment of realization. These are concrete
and specific (satisfying the elaborative encoding requirement) but use a
vocabulary natural to code work.

### Scene format

```
Moment: {One-paragraph narrative reconstruction. WHO was involved, WHAT
was being worked on, WHERE in the codebase, WHEN relative to other events,
and WHY a decision was made or insight occurred. Must include at least one
concrete code-level detail (a function name, an error message, a file path)
that anchors the scene to the actual codebase.}

Timeline: {anchor} -> {what happened} [{temporal marker}]
Prior: {what came before, if relevant}
After: {what followed or is expected to follow, if known}

(Narrative reconstruction only. Not a code specification.)
```

### Scene quality rules

**Temporal anchor rule:** When the information has any time dimension, the
scene MUST include a Timeline entry with a concrete temporal marker:
commit hashes, PR numbers, sprint/release boundaries, relative sequencing,
incident timestamps, or session references.

**Codebase anchor rule:** Every scene MUST reference at least one concrete
code-level artifact from the conversation: a file path, function name,
class name, error message, or module name.

**Contextual binding rule:** For recurring topics, capture WHY this came up
now -- what problem, symptom, or conversation triggered it.

**Scene validation:** No invented specifics. Do not include code artifacts
not present in the original conversation.

---

## 7. Retrieval Protocol

The three-state protocol, adapted to search both Letta Code's existing
blocks and the dual-trace additions.

### State A -- Fact + scene found

1. Read the fact (from dedicated file or existing Letta Code block)
2. Read the scene from scenes/{category}/{anchor}.md
3. Reconstruct the moment before formulating the answer
4. Answer with high confidence

### State B -- Fact found, no scene

1. Answer from the fact (existing Letta Code block or dedicated fact file)
2. Answer with medium confidence
3. Do not fabricate a scene

### State C -- Nothing found

1. Do not guess or confabulate
2. Respond: "I don't have that stored in my memory."

### Aggregation queries

For questions requiring aggregation ("what decisions have we made?", "have
we seen this before?"):

1. Glob the relevant category directory AND check existing Letta Code blocks
2. Read all matching files
3. For each fact with a linked scene, read the scene
4. Use Timeline fields to sequence entries chronologically
5. Synthesize across all entries before answering

### Staleness check (new for code)

If a fact or scene references code artifacts (file paths, function names)
that may have changed, note the staleness: "I have this stored from [date],
but the referenced files may have changed since then."

---

## 8. Auto-Activation

### Coexistence with Letta Code's memory check reminder

Letta Code already injects periodic reminders for the agent to check whether
the conversation contains learnable information. This handles routine fact
updates (preferences, basic conventions, project details).

The coding-memory-protocol.md directive adds a separate trigger layer for
scene-worthy events. The two mechanisms coexist:

```
Memory check reminder (Letta Code native):
  -> "Should I update human.md or conventions.md?"
  -> Handles routine fact updates

Coding-memory-protocol.md (dual-trace skill):
  -> "Is a design decision being made? Is a bug being resolved?"
  -> Triggers FULL encoding with scene when detected
```

The directive is pinned in system/ so it's in context on every turn,
alongside the existing Letta Code system files.

### Trigger list

See references/coding-memory-protocol-template.md for the complete
directive. Summary:

**Encode (invoke skill):**
- Design decisions with rationale
- Debugging incidents resolved
- Conventions established with origin stories
- Learning progressions observed
- Recurring patterns (second+ occurrence)

**Skip (let Letta Code handle normally):**
- Routine edits, transient debugging, basic preferences
- Information already stored

---

## 9. Encoding Workflow

### Step 1: Check existing memory

Before writing anything, check whether Letta Code already has this
information stored:

```
Read system/{project}/conventions.md    -- for conventions
Read system/human.md                    -- for preferences
Read decision/{anchor}.md               -- for decisions
Read debug/{anchor}.md                  -- for incidents
```

If the fact already exists in a Letta Code block: don't duplicate it. Just
add a scene (if FULL) that links back to the existing location.

### Step 2: Score evidence (0-12)

Apply the four dimensions (Section 4). Route: SKIP / STREAMLINED / FULL.

### Step 3: Write fact (if needed)

If the information fits naturally into an existing Letta Code block
(conventions.md, human.md), update that block using the Memory tool.

If the information needs its own file (decisions with complex rationale,
incidents, learning progressions, patterns), write to the appropriate
category directory using the Write tool.

### Step 4: Write scene (FULL path only)

Write to scenes/{category}/{anchor}.md using the Moment format (Section 6).
The `linked_fact` field points to wherever the fact lives -- whether that's
a dedicated file or an existing Letta Code block path.

### Step 5: Commit

```bash
git -C "$MEMORY_DIR" add .
git -C "$MEMORY_DIR" commit -m "mem: store {category}/{anchor} ({type}, evidence:{score})"
git -C "$MEMORY_DIR" push
```

---

## 10. What's New vs. What's Inherited

| Component | Source | Notes |
|---|---|---|
| Git-backed memfs | Letta Code (existing) | Used as-is |
| system/ pinned files | Letta Code (existing) | Directive added here |
| human.md | Letta Code (existing) | Preferences continue to live here |
| {project}/conventions.md | Letta Code (existing) | Simple conventions stay here |
| Memory tool (str_replace, etc.) | Letta Code (existing) | Used for updating existing blocks |
| /init bootstrapping | Letta Code (existing) | No changes |
| /remember command | Letta Code (existing) | No changes |
| Memory check reminder | Letta Code (existing) | Coexists with skill triggers |
| scenes/ directory | **New** | All scene files |
| decision/ directory | **New** | Structured decision records |
| debug/ directory | **New** | Structured incident records |
| learning/ directory | **New** | Developer skill progressions |
| pattern/ directory | **New** | Recurring pattern observations |
| Evidence scoring (0-12) | **New** | Determines encoding depth |
| Scene Moment format | **New** | Narrative reconstruction + Timeline |
| Three-state retrieval (A/B/C) | **Adapted from personal-memory** | Searches both Letta Code blocks and new files |
| STREAMLINED / FULL routing | **Adapted from personal-memory** | SKIP replaces DROP (Letta Code handles baseline) |
| Scene quality rules | **Adapted from personal-memory** | Temporal, codebase, and binding rules |
| Auto-activation directive | **Adapted from personal-memory** | Coexists with memory check reminder |

---

## 11. Open Design Questions

### 11.1 Scene length for incidents

Debugging incidents can be complex narratives. Recommendation: soft cap
at ~150 words, hard cap at ~250 words. Split complex incidents into
multiple linked entries.

### 11.2 Automatic pattern detection

Should the agent periodically review recent entries for recurring themes?
Could be a scheduled task or manual "/review-patterns" command. Deferred
to implementation.

### 11.3 Staleness detection

Code changes faster than personal facts. Recommendation: add optional
`review_by` field to frontmatter; implement path-existence checks at
retrieval time.

### 11.4 Integration with git history

Should encoding pull commit messages or PR descriptions as evidence?
Recommendation: allow but don't require. Include commit references when
mentioned in conversation. Don't auto-scrape git history.

### 11.5 Memory tool vs. Write tool boundary

The Memory tool (str_replace, insert, create, delete, rename) is Letta
Code's native mechanism for updating blocks. The Write tool creates new
files. The skill uses both: Memory tool for existing blocks, Write for
new scene and fact files. This boundary should be validated in practice
-- there may be edge cases where one tool is more appropriate than
expected.

### 11.6 Interaction with /remember

When a developer says "/remember that we switched to gRPC for internal
calls because of serialization overhead," should the dual-trace skill
intercept this and add a scene, or should /remember handle it normally
and the skill only activate for non-explicit memory events? Recommendation:
let /remember handle the fact storage, then check whether the content
warrants a scene (if rationale is present, score and potentially add a
scene file). This avoids conflicting with /remember's existing behavior.

### 11.7 Retrieval framing mismatch (encoding specificity)

Tulving & Thomson (1973) showed that retrieval fails when the cue at
recall does not match the context present at encoding -- even when the
cue is a strong semantic associate of the target. The current retrieval
protocol is anchor-dependent: the agent derives an anchor term from the
developer's question and searches by that term. If the question is framed
differently from the encoding anchor ("why is our serialization format
binary?" when the scene is filed as grpc-migration), the Glob search
returns nothing even though the scene's content is directly relevant.

Recommendation: add a content-search fallback. If anchor-based Glob
returns no results, Grep across all scene files in the relevant category
for key terms from the question. This catches framing mismatches without
requiring a question-type taxonomy. A more complete solution -- tagging
each scene with multiple retrieval cues at encoding time, or indexing
scenes by the type of question they answer -- is worth considering as
volume grows but adds encoding overhead.

### 11.8 Distinctiveness erosion and scene consolidation

Fernandes et al. (2018) showed that the drawing advantage is partly
selectivity-driven: drawing is effortful, so drawn items are distinctive
relative to the broader encoding context. If the agent encodes too many
scenes, individual scenes lose their distinctiveness advantage and begin
interfering with each other, analogous to proactive interference in
human memory.

The evidence scoring threshold (>= 8 for FULL, or >= 5 with stakes
override) provides the primary protection against volume inflation.
Coding sessions are also naturally lower-volume than personal memory
contexts: architectural decisions, significant incidents, and learning
progressions are genuinely sparse relative to routine coding activity.

For longer-lived agents, a consolidation mechanism becomes worthwhile.
Two options in increasing complexity:

1. Hard cap per category (e.g., 20-30 scenes per category): when
   approaching the cap, surface a prompt for the developer to review and
   retire low-value entries before encoding new ones.

2. Periodic review pass (analogous to sleep consolidation): a scheduled
   or manual "/review-scenes" command that scans all scenes, flags entries
   with no recorded retrieval history after N sessions, and marks them
   for retirement or archival. Requires tracking retrieval events (a
   `last_retrieved` field in frontmatter).

Option 1 is a reasonable near-term proxy. Option 2 is the architecturally
correct long-term solution but requires retrieval event instrumentation
not currently in the system.

---

## 12. Relationship to Personal-Memory System

The coding dual-trace system coexists with the personal-memory system.
Both share the same core principles (dual-trace encoding, three-state
retrieval, scene quality rules) but differ in:

- Scoring dimensions (personal disclosure vs. code decisions)
- Scene vocabulary (visual metaphors vs. narrative reconstruction)
- Trigger lists (personal information vs. coding events)
- Storage integration (standalone vs. layered on Letta Code memfs)

A future unification could merge both into a single skill with
context-dependent behavior, but the cleaner starting point is two
independent skills with shared design principles.
