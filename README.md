# Drawing on Memory for Coding Agents

A dual-trace memory skill for Letta Code agents. Companion to
`sternb12/agent_draw_skills`, which documents the parent protocol and its
evaluation on LongMemEval-S.

The primary skill in this repo is the coding adaptation described in Section 5
of the manuscript. Cloning to `.skills/drawing-memory` installs it directly
(SKILL.md lives at repo root, which is what Letta Code's skill discovery
expects).

A file-based port of the original personal-memory protocol is also preserved
here under `personal-memory/` for reference. It is not installed by the
default clone command; see `personal-memory/README.md` if you want it.

If you are looking for the personal-memory skill evaluated in the manuscript,
that version runs on Letta v1 archival memory and lives at
`sternb12/agent_draw_skills`. The approach in this repo adapts that same
protocol to file-based storage.

---

## Why a separate skill for coding?

The personal-memory protocol was designed around the kinds of things a tutor
or assistant wants to remember about a person: preferences, constraints,
episodes with enough concrete detail that the later retrieval cues feel
natural. Coding sessions produce a different mix. Most of what happens during
a session is transient, where a file gets renamed, an import gets reordered, a
wandering `print` gets removed, and it's not helpful in most cases to
encode that information at the same depth as an architectural decision. At the
other end of the distribution there are a smaller set of events that really should
survive the session: the reason we picked Postgres over Redis, the incident
where the auth middleware started throwing 401s under load, the convention
that all service modules expose a `health()` endpoint.

The challenge is that a single encoding policy will probably struggle to
serve both ends of this distribution well. If we encode everything, the
memory store fills up with noise, which impacts retrieval. If we encode nothing
without explicit instruction, the agent forgets the things we want it to remember.
What we want is a routing decision made at encoding time, on the agent's
own initiative that skips the noise, records a compact fact, and writes a
full fact-plus-scene pair only for the meaningful events whose later
retrieval will be worth that extra effort.

Section 5 of the manuscript illustrates this adaptation. This repository is a
working implementation.

---

## How the coding skill differs from the personal-memory skill

The parent protocol scores evidence on three dimensions (Relevance,
Specificity, Explicitness) and routes to one of two tiers (DROP or FULL).
The coding skill keeps the dual-trace idea and the three-state retrieval
protocol intact, but changes three things:

**Four scoring dimensions, tuned to what matters in code.** Durability
(will this be true tomorrow, next week, next year?), Scope (one line, one
module, the whole system?), Rationale rich (do we have the 'why' or
just the 'what'?), and Retrieval likelihood (does anyone really care or
will anyone ever ask about this again?). Each dimension has a 0-3 anchor,
so the total ranges 0-12.

**Three tiers instead of two.** SKIP (0-4) defers to Letta Code's existing
memory behavior and doesn't write anything new. RECORD (5-7) writes a compact fact
file only. FULL (8-12) writes the fact file and a scene file. An override
decision promotes incidents (outages, data loss, security issues) and
irreversible decisions (migrations, API contracts, framework switches) to
FULL once they pass score 5, with the thinking that the cost of an extra scene
file is small and the cost of missing an impactful incident is not.

**Six information types, each with its own folder.** `decisions/`,
`incidents/`, `conventions/`, `patterns/`, `learning/`, and `preferences/`.
Conventions and preferences are special: Letta Code already has a permanent
home for them in the `system/` memory block, so the skill writes only the
scene and links back to the block rather than duplicating a fact.

The scene trace carries a `[MOMENT:anchor]` tag (instead of the
`[SCENE:anchor]` tag of the personal-memory version) and includes inline
Timeline, Prior, and After fields. The footer on every scene file,
`(Mnemonic reconstruction only. Not a code specification.)`, is there to
make it unambiguous that the scene exists to support recall later. It's
not there to define behavior the code should actually have.

---

## Pilot validation

The coding skill was tested on a single running Letta Code agent
(`agent-34afd9...`) on 3/24/2026. Four tests, all PASS. These are
sanity checks, not an evaluation. The goal was to confirm that the
routing, retrieval, and update behaviors described in Section 5 actually
functioned on a live agent before investing in a larger run.

**Test 1 -- Routing discrimination.** Two messages in sequence. A simple
preference ("I like to keep my functions under 30 lines") routed SKIP and
landed in `human.md` with no scene file. An architectural decision
(switching to JSON structured logging because Datadog could not parse
free-form strings, with the rejected alternative of a log shipper) routed
FULL: the agent wrote `decisions/structured-logging-json.md` with an
evidence score of 11 and a matching scene file in `scenes/decisions/`.

**Test 2 -- State A retrieval.** Asked the agent *"Why did we switch to
JSON structured logging?"* in the same session. The agent located both the
fact file and the scene file, labeled its own retrieval state as "STATE A
retrieval: fact + scene, high confidence," and quoted verbatim
from the scene's rationale ("adding complexity to fix a problem we'd
created ourselves") while noting Datadog's parsing failures as the cause.

**Test 3 -- State C abstention.** Asked *"What's our test coverage
threshold?"* -- it's plausible-sounding but with an unstored answer. The agent
responded *"I don't have that stored in my memory,"* labeled its retrieval
state as "STATE C retrieval: no fact found, abstaining," did not guess a
number, and offered to store one if provided. This is appropriate behavior
because: failure to retrieve should produce silence, not confabulation.

**Test 4 -- Update, not duplicate.** A two-session test. In Session 1 the
agent was told about an auth middleware intermittently throwing 401 errors in
staging, suspected root cause a race condition in the token refresh. The
agent scored the event 8 (stakes override also applied, since auth
incidents qualify as security issues), wrote `incidents/auth-middleware-
401-race.md` with confidence MEDIUM, and paired it with a scene. In
Session 2 the same agent was told the issue had recurred on a different
endpoint and the root cause was confirmed: the token refresh function was
not idempotent and two concurrent requests were both trying to refresh.
The agent found the existing file, upgraded its confidence from MEDIUM to
HIGH, and extended the scene's Timeline field rather than writing a
second file. After both sessions the `incidents/` directory contained
exactly one file.

All four tests, together with the exact messages, file contents, scores,
and commit messages, are in `drawing-memory/PILOT_TESTS.md`.

---

## A caveat worth reading before installing

The pilot tests uncovered a failure that was not in the skill itself
but in how we attempted to load skills to Letta Code. On the first two
attempts, the agent received a message that clearly described an
architectural decision, correctly recognized it as architectural, and then
routed it to `human.md` anyway, because Letta Code's default memory
behavior (update `human.md`) triggered before the skill-loading step was
triggered. The directive in `system/` pointing to the skill was not
sufficient to override the default on a clean start.

The workaround used in the pilot is simple and it worked for every single
subsequent input in the session: ask the agent to explicitly load the
skill once at the start of the session. After that single invocation the
agent internalized the routing and the skill fired automatically for
every scene-worthy input that followed. This was the only validated setup
path, and it is the one the installation instructions describe below.

A cleaner fix is in progress. The v2 design (`references/
coding_dual_trace_v2.md`) moves encoding out of the conversation entirely
and runs it as a post-session sub-agent fired from a Stop hook -- a
pattern borrowed from Claude Code's autoDream. That design eliminates the
skill-loading indirection by not requiring the skill to fire
mid-conversation at all. The v2 work is planned but not yet tested.

---

## Installation

```
git clone https://github.com/sternb12/letta-code-draw-skill .skills/drawing-memory
```

Enable memfs (/memfs enable) for your agent and run `/init`. At the start of each coding
session, ask the agent once:

> "Please load your drawing-memory skill."

After that one-time invocation the skill will be called automatically on
scene-worthy inputs for the rest of the session. This limitation is described above.

---

## Parent protocol and evaluation

The personal-memory protocol this skill is adapted from was evaluated on
LongMemEval-S (Wang et al., 2024) against a fact-only control. The headline
result: 73.7% vs 53.5% overall accuracy, a gain of 20.2 percentage points,
with the largest gains on multi-session, knowledge-update, and
temporal-reasoning questions. Full results, the evaluation harness, and the
original personal-memory skill live in the parent repository:

  https://github.com/sternb12/agent_draw_skills

The coding skill in this repository has not been run through
LongMemEval-S or any comparable benchmark. Section 5 of the manuscript
describes the adaptation and the pilot tests above; a larger evaluation
is future work.

---

## Research citation

Fernandes, M. A., Wammes, J. D., & Meade, M. E. (2018). The surprisingly
powerful influence of drawing on memory. *Current Directions in
Psychological Science*, 27(5), 302-308.
https://doi.org/10.1177/0963721418755385
