# memory-protocol.md -- System Prompt Directive Template
#
# PURPOSE: Copy this content into the agent's memfs at:
#   $MEMORY_DIR/system/memory-protocol.md
#
 Files in system/ are pinned to the agent's context on every turn.
 This directive ensures reliable skill auto-activation without
 requiring the user to explicitly say "use your drawing-memory skill."

  SETUP: After creating a new agent, write this file and commit:
   git -C "$MEMORY_DIR" add system/memory-protocol.md
   git -C "$MEMORY_DIR" commit -m "feat: add drawing-memory auto-activation directive"
   git -C "$MEMORY_DIR" push

# ---------------------------------------------------------------
# PASTE EVERYTHING BELOW THIS LINE INTO system/memory-protocol.md
# ---------------------------------------------------------------

---
description: Drawing-memory activation rule -- when to invoke the
  drawing-memory skill to encode and retrieve personal information.
---

# Memory Protocol

## When to Invoke Drawing-Memory

ALWAYS invoke the drawing-memory skill before responding conversationally
when a user shares any of the following:

  - Name, age, location, or biographical details
  - Occupation, workplace, specialization, or career history
  - Health information (conditions, providers, medications, history)
  - Relationships (family, partner, close friends, colleagues)
  - Preferences (food, hobbies, routines, interests)
  - Events (appointments, milestones, things that happened)
  - Changes or updates to previously shared information
  - Any information the user explicitly asks you to remember

Do not wait for the user to say "remember this" or "use your
drawing-memory skill." If the information meets the above criteria,
invoke the skill immediately, then respond conversationally.

## How to Invoke

1. Call the Skill tool: Skill(command="load", skill="drawing-memory")
2. Follow the encoding workflow in the loaded SKILL.md
3. After encoding and committing, call: Skill(command="unload", skill="drawing-memory")
4. Then respond conversationally

## When to Invoke for Retrieval

When the user asks a question that may be answerable from stored memory
(e.g., "what do you know about me?", "do you remember what I do for
work?", "which of my hobbies did I start first?"):

1. Check $MEMORY_DIR/facts/ for relevant files before answering
2. Follow the three-state retrieval protocol in SKILL.md
3. Do not guess or confabulate if no file is found

## When NOT to Invoke

  - General world knowledge questions
  - Transient conversational exchanges ("thanks", "ok", "got it")
  - Information the user explicitly marks as temporary or speculative
  - Information already confirmed as stored (avoid redundant encoding)
