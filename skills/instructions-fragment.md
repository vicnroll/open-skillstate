<!-- BEGIN skillstate -->
## Execution continuity

- For non-trivial repository work — multiple tool calls, edits, debugging or test cycles, follow-up turns, context compaction, or multiple sessions — use the `skill-state` skill autonomously. Do not wait for the user to ask for it.
- Before substantial work, run `skillstate get`. If the state matches the user's objective, continue from `next_action`; if the request starts or supersedes that objective, reinitialize it.
- Change state only through `skillstate patch`. Never edit `.skillstate/state.json` directly: the CLI owns the schema, and direct writes are detected.
- Patch at meaningful checkpoints and before finishing a work block. Keep the state as compact current execution state, never a transcript. Never store chain-of-thought, secrets, or raw tool output.
- Set `mode` to `exploration` when debugging, auditing or exploring, and back to `execution` once the objective is concrete.
- The user's latest instruction, the current repository and current tool results are authoritative. Correct stale state rather than following it.
- Routine bookkeeping is silent. Do not report state reads or writes unless asked, or unless the state reveals a problem affecting the task.
- Simple explanatory Q&A and tiny atomic tasks need no state.
<!-- END skillstate -->
