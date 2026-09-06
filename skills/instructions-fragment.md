<!-- BEGIN skstate -->
## Execution continuity

- For non-trivial repository work — multiple tool calls, edits, debugging or test cycles, follow-up turns, context compaction, or multiple sessions — use the `skstate` skill autonomously. Do not wait for the user to ask for it.
- Before substantial work, run `skstate get`. If the state matches the user's objective, continue from `next_action`; if the request starts or supersedes that objective, reinitialize it.
- Change state only through `skstate patch`. Never edit `.openskillstate/state.json` directly: the CLI owns the schema. Where the client supports it, direct edits are blocked; elsewhere an external change is validated on the next CLI operation.
- Patch at meaningful checkpoints and before finishing a work block. Keep the state as compact current execution state, never a transcript. Never store chain-of-thought, secrets, or raw tool output.
- Set `mode` to `exploration` when debugging, auditing or exploring, and back to `execution` once the objective is concrete.
- Record verification checks only; `verification.overall` is derived by the runtime.
- The user's latest instruction, the current repository and current tool results are authoritative. Correct stale state rather than following it.
- Routine bookkeeping is silent. Do not report state reads or writes unless asked, or unless the state reveals a problem affecting the task.
- Simple explanatory Q&A and tiny atomic tasks need no state.
<!-- END skstate -->
