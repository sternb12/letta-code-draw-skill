# Coding Dual-Trace Memory v2: Post-Session Stop Hook Architecture

**Date:** April 2026
**Status:** Design sketch
**What changed from v1:** Moves encoding from inline/mid-conversation (v1) to
post-session forked sub-agent (v2), borrowing the autoDream pattern from the
Claude Code source code (Anthropic, 2026).

---

## The v1 Problem This Solves

Pilot testing of v1 (March 2026) identified a clean failure mode: the agent
correctly identifies when a coding event is scene-worthy ("this is an
architectural decision with rationale") but routes to human.md anyway because
Letta Code's default memory behavior overrides the protocol directive. Even
with an IMPORTANT block explicitly prohibiting human.md for decisions, the
default wins.

The root cause is architectural: we are asking the agent to interrupt its own
coding flow mid-conversation to run a multi-step encoding workflow (score ->
write fact -> write scene -> git commit). That interruption fights the grain of
how the agent operates. The autoDream pattern from Claude Code's source suggests
a cleaner alternative: move encoding out of the conversation entirely and fire
it as a post-session background pass over the complete transcript.

---

## Core Architecture

```
[Coding session ends]
        |
        v
[Stop hook fires]
        |
        v
[Three-gate check] --------> SKIP (no gates pass): nothing written
        |
   all gates pass
        |
        v
[Encoding sub-agent receives full transcript]
        |
        v
[Evidence gate: SKIP / RECORD / FULL per event]
        |
        v
[Write fact files and/or scene files to $MEMORY_DIR]
        |
        v
[Git commit: mem: store {anchor} ({type}, evidence:{score})]
        |
        v
[Return summary to parent agent]
```

---

## Three-Gate Cheapest-First Filter

Borrowed from autoDream (Claude Code, KAIROS_DREAM subsystem). Run gates in
order; abort as soon as one fails. The goal is to avoid running the expensive
encoding sub-agent on sessions that contain nothing worth storing.

**Gate 1 -- Activity check (cheapest, no LLM call):**
Did the session involve any file edits, commits, or tool calls beyond
conversation? If the session was pure Q&A with no Write/Edit/Bash calls, skip.

**Gate 2 -- Signal check (lightweight scan):**
Does the transcript contain any of these signals?
- Explicit decision language: "we decided", "we're switching", "instead of X
  we'll use Y", "the reason is", "tradeoff"
- Debugging resolution arc: error message followed by a fix followed by a
  passing test or confirmation
- Convention being established: "from now on", "always", "never", "the rule is"
- 3+ file edits in a single session (concrete event trigger, from VERIFICATION_AGENT
  pattern -- no subjective scoring needed)

If none of these signals appear, skip.

**Gate 3 -- Lock check (prevent concurrent encodes):**
Is another encoding pass already running for this agent? Check for a lockfile
at $MEMORY_DIR/.encode_lock. If locked, skip this session (it will be picked
up in the next pass).

---

## What the Encoding Sub-Agent Receives

The sub-agent gets a structured prompt containing:

1. **Full session transcript** -- all tool calls and responses, not just the
   final state. The arc matters: what was tried before the solution, what
   errors appeared, what was rejected.

2. **Existing index** -- a compact listing of anchors already in $MEMORY_DIR
   (decision/, debug/, convention/, etc.) so the sub-agent can detect updates
   to existing entries vs. new entries.

3. **Instructions** -- the evidence gate rubric, the six information types,
   the Moment: scene format, and the three-state retrieval protocol.

The sub-agent does NOT receive the coding task context beyond the transcript.
It is a dedicated encoding agent, not a coding agent.

---

## Evidence Gate (unchanged from v1, 0-12 scale)

Score each scene-worthy event on four dimensions (0-3 each):

**Durability** -- how long will this matter?
  0 = this session only (a one-off workaround)
  1 = this sprint / this feature
  2 = this project long-term
  3 = affects all projects / permanent convention

**Scope** -- how much of the codebase does it affect?
  0 = one function
  1 = one file or module
  2 = multiple modules or an API boundary
  3 = architectural -- affects how everything fits together

**Rationale-richness** -- was reasoning stated?
  0 = bare decision, no reasoning ("we used X")
  1 = some reasoning ("we used X because it was easier")
  2 = reasoning + alternative considered ("we used X over Y because...")
  3 = full tradeoff: reasoning + alternatives + consequences stated explicitly

**Retrieval likelihood** -- how often will this be needed?
  0 = unlikely to ever search for this again
  1 = might search once or twice
  2 = will definitely come back to this
  3 = this is the kind of thing that gets forgotten and causes bugs

**Routing:**
  0-4:  SKIP -- let Letta Code handle natively or don't encode
  5-7:  RECORD -- write fact file only (no scene)
  8-12: FULL -- write fact file + scene file

**Concrete event overrides (from VERIFICATION_AGENT pattern):**
  If 3+ file edits occurred in the session: minimum RECORD regardless of score
  If error -> fix -> passing test arc is present: minimum FULL regardless of score
  If a PR was merged or a commit was made: minimum RECORD regardless of score

---

## Six Information Types and Their Anchors

**decision/** -- Architectural or design choices with stated rationale
  Anchor format: decision/{technology-or-pattern}
  Example: decision/session-storage-migration, decision/auth-approach
  Trigger: "we decided", explicit tradeoff, rejected alternative

**debug/** -- Bug investigations with identified root causes
  Anchor format: debug/{error-or-symptom}
  Example: debug/websocket-reconnect-failure, debug/empty-tool-result-crash
  Trigger: error -> investigation -> root cause -> fix arc

**convention/** -- Rules that emerged from specific experience
  Anchor format: convention/{rule-name}
  Example: convention/no-sed-for-edits, convention/anchor-naming
  Trigger: "from now on", "always", "never", agreement on a rule

**pattern/** -- Recurring developer behaviors (second+ occurrence)
  Anchor format: pattern/{behavior-name}
  Example: pattern/forgets-to-pull-before-push
  Trigger: second+ time the same issue appears across sessions

**learning/** -- Skill development and conceptual breakthroughs
  Anchor format: learning/{concept}
  Example: learning/git-rebase-vs-merge, learning/kv-cache-mechanics
  Trigger: explicit "I didn't know", "now I understand", concept clarification

**preference/** -- Workflow habits with an origin story
  Anchor format: preference/{habit}
  Example: preference/no-unicode-in-repos, preference/ask-before-commit
  Trigger: stated preference with reason ("I prefer X because Y")

---

## File Format

**Fact file** ($MEMORY_DIR/{type}/{anchor}.md):

```
---
description: {one-line summary}
type: {decision|debug|convention|pattern|learning|preference}
linked_scene: scenes/{type}/{anchor}.md
evidence_score: {0-12}
session_date: {YYYY-MM-DD}
---

{Third-person factual summary. Preserve the specific values, names, file paths,
and error messages that were present. Do not paraphrase into generalities.}

Alternatives considered: {if applicable}
Outcome: {what was decided and why}
```

**Scene file** ($MEMORY_DIR/scenes/{type}/{anchor}.md):

```
---
linked_fact: {type}/{anchor}.md
---

Moment: {Concrete narrative anchor. What was visible on screen -- which file
was open, what error message appeared, what the test output showed. Embed
the key information as specific imageable detail, not abstract description.
No invented specifics -- only what the transcript confirms.}

Timeline: {Where this sits in the project arc. What had just been attempted.
What had failed or been rejected before this moment.}

Prior: {What was in place before. What assumption was being made. What the
code looked like before the change.}

After: {What this enables. What the next step became. What was unblocked.}

(Coding scene only. Not evidence. Specifics confirmed from session transcript.)
```

---

## Why Post-Session Produces Better Scenes

A mid-conversation agent writing a scene sees only the moment a decision was
mentioned. A post-session agent sees the full arc:

- What was tried before the solution (the Prior field)
- What errors appeared along the way (the Timeline field)
- How the conversation resolved (the After field)
- Whether the decision was revisited or confirmed later in the session

The Claude Code compaction prompt requires verbatim quotes specifically to
prevent "intent drift" -- after summarization, the agent subtly reinterprets
what was asked. Scene traces solve the same problem for cross-session memory:
committing to concrete specifics (the error message, the file name, the test
output) prevents the fact from drifting toward a generic summary over time.

Post-session encoding means every scene is written with full information,
not partial context. This is expected to produce richer temporal anchors
and more reliable Prior/After fields.

---

## Three-State Retrieval (unchanged from v1)

When the developer asks a question that may benefit from stored context:

**State A** -- fact file AND scene file found for anchor:
  High confidence. Read both files. Reconstruct the Moment as context.
  Answer draws on the specific details in the scene.

**State B** -- fact file only (no scene):
  Medium confidence. Answer from fact file. Note that scene context is
  unavailable (RECORD-tier encoding, no temporal anchor stored).

**State C** -- no file found for anchor:
  Abstain. Do not guess. "I don't have that stored."
  Do not draw on general knowledge to fill the gap.

**Multi-session (aggregate) questions:**
  Glob all files in the relevant type directory. Use scene Timeline fields
  to sequence events. Use Prior/After fields to track how a decision evolved.

---

## Comparison: v1 vs v2

| Dimension | v1 (inline) | v2 (post-session) |
|---|---|---|
| Trigger | Agent recognizes event mid-conversation | Stop hook fires after session ends |
| Information available | Partial (mid-conversation) | Full transcript |
| Scene quality | Based on current context only | Full arc: Prior, Timeline, After |
| Fight with defaults | Yes -- overrides Letta Code defaults | No -- separate dedicated pass |
| Auto-activation | Broken in pilot | Not needed -- always fires via hook |
| Routing | Evidence score only | Evidence score + concrete event triggers |
| Latency | Zero (inline) | Small delay after session |
| Implementation | Letta Code skill + system prompt directive | Stop hook + encoding sub-agent |

---

## Implementation Path

**Option A -- claude-subconscious Stop hook (cleanest):**
  Use the claude-subconscious hook pattern (Anthropic, 2026). After each
  response, the Stop hook fires and sends the full transcript to a dedicated
  Letta encoding agent. The encoding agent runs the three-gate check, scores
  events, and writes files. The parent coding agent continues uninterrupted.

**Option B -- Manual encode command:**
  Add a /encode slash command that the developer runs at the end of a session.
  Less automatic but requires no hook infrastructure. Useful for initial testing.

**Option C -- Session-end trigger:**
  Detect session end via idle timeout (no new messages for N minutes) and
  trigger encoding automatically. Middle ground between A and B.

For initial validation: start with Option B (manual /encode) to confirm the
encoding sub-agent produces correct output, then move to Option A for full
automation.

---

## Open Questions

1. **Encoding agent identity**: Should the encoding agent be the same Letta Code
   agent (in a new message thread) or a separate lightweight agent? Using the
   same agent preserves existing memory context; a separate agent is cleaner
   but requires bootstrapping.

2. **Transcript length**: Full transcripts for long sessions may be very large.
   Should the gate-2 signal check pre-filter to only the relevant segments
   (tool calls + surrounding messages) before sending to the encoding agent?

3. **Update vs. insert**: When a new session updates an existing anchor (e.g.,
   a decision is revisited), should the encoding agent overwrite the fact file
   or append a dated update? Appending preserves history for knowledge-update
   retrieval; overwriting keeps the fact file clean.

4. **Pilot design**: To properly compare v1 vs v2, need the same set of coding
   sessions encoded both ways. The LME-S precedent (C6 vs C7 controlled
   comparison) suggests running v2 with and without scene traces on the same
   session set, then testing retrieval on the same question set.

---

## Connection to Personal Memory Results

The three LME-S categories where dual-trace showed the largest gains map to
exactly the coding memory scenarios where post-session encoding is expected to
help most:

| LME-S category | Personal gain | Coding equivalent |
|---|---|---|
| Knowledge-update | +25 pp | "Why did we switch from X?" -- decision was revised |
| Temporal-reasoning | +40 pp | "What did we try before settling on Y?" -- Prior field |
| Multi-session | +30 pp | "Have we seen this error before?" -- debug pattern across sessions |

Single-session (0 pp gain) maps to: "What does this function do?" -- a lookup
that doesn't need scene context. The effect is expected to be specific to the
same categories.

---

## References

Anthropic (2026). Claude Code TypeScript source (accidentally shipped in
@anthropic-ai/claude-code npm package, March 31, 2026). Relevant subsystems:
autoDream (KAIROS_DREAM), VERIFICATION_AGENT feature flag, compaction verbatim-
quote requirement, three-tier context pipeline.

Dibia, V. (2026, April 6). Inside Claude Code. Designing with AI, Issue 62.

Fernandes, M. A., Wammes, J. D., & Meade, M. E. (2018). The surprisingly
powerful influence of drawing on memory. Current Directions in Psychological
Science, 27(5), 302-308.

Stern, B. (2026). Coding dual-trace memory v1: pilot test results. PILOT_TESTS.md.
