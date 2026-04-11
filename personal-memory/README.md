# Drawing-Memory Skill: Personal Memory Variant

A file-based port of the personal-memory dual-trace protocol to Letta Code's
memfs. This variant stores fact and scene traces as flat markdown files in
the agent's git-backed memory directory, using Write + Glob rather than
archival_memory_insert + archival_memory_search.

This is a preserved variant, not the primary install target of this repo.

The primary skill in this repo is the coding adaptation at the root level
(SKILL.md). Clone the repo to install that skill:

  git clone https://github.com/sternb12/letta-code-draw-skill .skills/drawing-memory

To install this personal-memory variant separately, copy it by hand:

  cp -r .skills/drawing-memory/personal-memory/ .skills/personal-memory/

For the evaluated personal-memory protocol (Letta v1 archival memory
condition, LongMemEval-S results), see the parent repository:

  https://github.com/sternb12/agent_draw_skills
