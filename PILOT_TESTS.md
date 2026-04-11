# Pilot Tests: Coding Dual-Trace Memory

Small manual tests to validate core behaviors before any larger evaluation.
Run via TUI. Verified by inspecting memfs after each test.

Agent: agent-34afd980-0d29-4015-a5e8-ec9112aea486
Date run: 2026-03-24

---

## Results Summary

| Test | Result | Files created | Notes |
|---|---|---|---|
| 1A routing (SKIP) | PASS | human.md updated | Preference correctly routed to human.md |
| 1B routing (FULL) | PASS | decision/ + scenes/decision/ | Requires explicit skill activation first (see findings) |
| 2 State A retrieval | PASS | n/a | Scene phrase quoted verbatim in answer |
| 3 State C abstention | PASS | n/a | Clean abstain, no guessing, offered to store |
| 4 Update (not duplicate) | PASS | debug/ updated in place | Confidence upgraded MEDIUM->HIGH, scene arc extended |

---

## Key Finding: Auto-Activation

Auto-activation via the system/ directive did not work reliably. The agent
defaulted to human.md for all inputs -- including architectural decisions --
on the first two attempts, despite the directive explicitly prohibiting this.

Root cause: the directive pointed to the skill ("invoke the skill when X")
but the skill-loading step was the indirection that failed. The agent's
trained default behavior (update human.md) overrode the directive before
the skill was ever invoked.

Resolution in pilot: asked the agent to explicitly invoke the skill once.
After that single explicit invocation, the agent internalized the routing
and auto-fired the skill correctly for all subsequent scene-worthy inputs
in the same session.

Open work: rewrite coding-memory-protocol.md to put the condensed encoding
workflow inline (eliminating the skill-load indirection), so auto-activation
works from the first message of a new session.

---

## Test 1: Routing Discrimination

**What it tests:** Agent correctly distinguishes SKIP (routine preference)
from FULL (scene-worthy decision) without being told which to apply.

**Method:** Two messages in sequence. Let agent respond fully to each.

Message A (expect SKIP):
> "I like to keep my functions under 30 lines. If it's getting longer I
> break it up."

Message B (expect FULL):
> "We're switching to JSON structured logging instead of plain text.
> The main driver was that our Datadog pipeline couldn't parse free-form
> strings reliably. We looked at keeping plain text and adding a log
> shipper to transform it, but that felt like adding complexity to fix
> a problem we'd created ourselves."

**Result: PASS**
- Message A: stored in human.md under Coding preferences. No scene file.
- Message B: decision/structured-logging-json.md created (evidence: 11);
  scenes/decision/structured-logging-json.md created. Agent quoted the
  exact phrase "adding complexity to fix a problem we'd created ourselves"
  from the scene in its response.

**Note on earlier attempts:**
Initial messages (Jest-to-Vitest, 2-space indentation) were used in a
failed first attempt before explicit activation. Those messages ended up
in conversation history, causing the agent to treat them as already-stored
on the second attempt. Switched to fresh messages (function length,
structured logging) which the agent had not seen.

---

## Test 2: State A Retrieval

**What it tests:** Agent finds both fact and scene, reconstructs moment,
answers with full context including rejected alternatives.

**Prerequisite:** Test 1 complete; decision/structured-logging-json.md
and scenes/decision/structured-logging-json.md exist.

**Method:** Ask in same session:
> "Why did we switch to JSON structured logging?"

**Result: PASS**
- Agent identified anchor, found fact file and scene file.
- Labeled its own retrieval state: "(STATE A retrieval: fact + scene,
  high confidence)"
- Answer included Datadog parsing failures as driver and exact quoted
  rejection rationale for log shipper.
- No confabulation.

---

## Test 3: State C Abstention

**What it tests:** Agent does not confabulate when asked about something
plausible but not stored.

**Method:** In same session as Test 2, ask:
> "What's our test coverage threshold?"

**Result: PASS**
- Agent responded: "I don't have that stored in my memory."
- Labeled its own retrieval state: "(STATE C retrieval: no fact found,
  abstaining)"
- Did not guess a percentage.
- Offered to store it if provided.

---

## Test 4: Update Behavior (Not Duplicate)

**What it tests:** On second occurrence of a related incident with new
information, agent updates the existing entry rather than creating a
duplicate.

**Note on design:** Original test expected STREAMLINED on Session 1, but
the agent correctly applied the stakes override rule: auth incidents
qualify as security issues, so score >= 5 routes FULL immediately.
STREAMLINED was never reached. Test was reframed to test update behavior
on an already-FULL entry.

**Session 1 message:**
> "The auth middleware is throwing 401s intermittently -- passes locally
> but fails in staging. Looks like a race condition in the token refresh."

**Session 1 result:**
- Agent scored evidence: 8 (Durability 2, Scope 2, Rationale-richness 1,
  Retrieval likelihood 3)
- Stakes override applied: auth incident >= 5 -> FULL immediately
- debug/auth-middleware-401-race.md created (confidence: MEDIUM,
  status: active investigation)
- scenes/debug/auth-middleware-401-race.md created
- Commit: "mem: store debug/auth-middleware-401-race (incident,
  evidence:8, status: active)"

**Session 2 message:**
> "The 401 issue is back, different endpoint this time -- the user
> session service. I confirmed the root cause by checking logs: two
> simultaneous requests both see an expired token and both try to
> refresh. The token refresh function isn't idempotent."

**Session 2 result: PASS**
- Agent found and updated the existing file. No duplicate created.
- debug/ contains exactly 1 file after both sessions.
- Confidence upgraded: MEDIUM -> HIGH (suspected -> confirmed root cause)
- Description updated to include user session service and "confirmed"
- Scene Timeline extended: two entries showing investigation arc
  (started -> root cause confirmed after log analysis)
- Commit: "mem: update debug/auth-middleware-401-race (root cause
  confirmed, fix pending)"

---

## Diagnostic: Self-Assessment Query

Before the routing tests, we asked the agent to describe its coding memory
protocol. It described Letta Code's built-in memory behavior accurately
but had no awareness of the coding-dual-trace-memory skill, the scenes/
directory, the decision/ directory, or the Moment format. It correctly
identified the MongoDB example as a durable architectural decision but
routed it to human.md.

This confirmed the failure was comprehension of the skill-loading step,
not recognition of scene-worthy events. The agent's event recognition
was correct throughout; only the routing destination was wrong.

Explicit invocation query used:
> "Please load your coding-dual-trace-memory skill and use it to encode
> this decision: we moved session storage from Redis to Postgres because
> Redis was adding operational overhead -- our session data is small
> enough that Postgres handles it fine, and keeping both systems seemed
> like the worst of both worlds."

The agent found SKILL.md via Bash(find . -name "SKILL.md"), read it,
correctly scored the evidence (12/12), applied stakes override, wrote
both files, and committed. After this single explicit invocation, the
skill fired automatically for all subsequent scene-worthy inputs.

---

## Open Work

1. Rewrite coding-memory-protocol.md to include condensed encoding
   workflow inline -- eliminates the skill-load indirection so
   auto-activation works from session start without explicit priming.

2. Clean up human.md -- contains stale Redis-to-Postgres bullet from
   earlier failed test runs; now superseded by dedicated decision file.

3. Test STREAMLINED path explicitly -- design a lower-stakes incident
   that does not trigger stakes override to validate the STREAMLINED
   routing and STREAMLINED-to-FULL upgrade pattern.
