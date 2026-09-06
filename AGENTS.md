<!--
Template note: merge the SKILL.state section below into the project's existing
AGENTS.md. Keep the project's architecture, build, test, style, review, and
other instructions alongside it. This file is not intended to replace them.
-->

## SKILL.state execution continuity

- For non-trivial repository work that may involve multiple tool calls, edits, debugging/test cycles, follow-up turns, context compaction, or multiple sessions, autonomously use the `skill-state` skill. Do not wait for the user to mention SKILL.state or ask them to opt in.
- Before substantial work on such a task, read `.skillstate/procedure.md` and `.skillstate/state.json`. If the current state matches the user's objective, continue it; if the latest request starts or supersedes the objective, reinitialize the state as defined by the procedure.
- Maintain `.skillstate/state.json` at meaningful checkpoints and before finishing a work block. Keep it as compact current execution state, not as conversation history; never store chain-of-thought, secrets, or unnecessary raw tool output.
- Treat the user's latest instruction, the current repository, and current tool results as authoritative. Correct stale state rather than following it.
- Routine SKILL.state bookkeeping is silent. Do not require the user to say "use SKILL.state", "read state", or similar wording.
- Simple explanatory Q&A or tiny atomic tasks that need no continuity do not require SKILL.state initialization.
