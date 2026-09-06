# SKILL.state execution procedure

This file defines the repository-level procedure for maintaining explicit execution state across long-running agent work.

The goal is to make conversational context disposable. The agent must preserve only the information required to continue the current task in `.skillstate/state.json`; the current repository and fresh tool observations remain the source of truth.

## 1. When to use this procedure

Use SKILL.state autonomously for non-trivial repository work that benefits from continuity, including work that is likely to involve any of the following:

- multiple meaningful tool calls;
- investigation followed by implementation or editing;
- debugging or repeated test/fix cycles;
- refactoring, migrations, or cross-file changes;
- work that may span multiple user turns, context compaction, or sessions;
- repository research whose conclusions are needed by later implementation steps.

Do not initialize SKILL.state for simple explanatory Q&A, a tiny atomic task that can be completed immediately, or work where no execution continuity is useful.

The user does not need to request SKILL.state explicitly. Detect when it applies and use it without asking for permission or requiring special wording.

## 2. Sources of truth

Use these sources in this order:

1. The user's latest instruction and explicit constraints.
2. The current repository/filesystem and current tool results.
3. `.skillstate/state.json` as the compact execution-continuity state.
4. Conversation history only as temporary context within the current session; do not depend on it for durable continuity.

If `state.json` conflicts with the repository or the user's latest instruction, the state is stale. Correct it immediately.

## 3. State lifecycle

`status` must be one of:

- `idle`: no active SKILL.state task;
- `active`: work is in progress;
- `blocked`: progress cannot continue without an external dependency or user decision;
- `completed`: the objective has been completed and verified to the extent required by the task.

### Starting a non-trivial task

Before substantial work:

1. Read `.skillstate/state.json`.
2. Decide whether the user's request continues the active objective or starts/supersedes it.
3. If there is no active matching objective, reinitialize the task-specific fields in `state.json` for the new objective.
4. Set `status` to `active` and set a concrete `next_action`.
5. Capture only constraints or acceptance criteria that are necessary for later steps.

Do not ask the user a bookkeeping question merely to decide whether to reuse or reset SKILL.state. Use the latest request and repository context to make that determination.

### Continuing an existing task

When `status` is `active` or `blocked` and the user's request is part of the same objective:

1. Read the current state before substantial work.
2. Reconcile it against the repository and current observations.
3. Incorporate new user constraints into the state when they affect future work.
4. Continue from `next_action`, changing it when new evidence requires a different path.

### Starting a different task

If the user clearly starts a different objective, the latest user instruction wins. Reinitialize the task-specific state instead of following the previous `next_action`.

SKILL.state is execution state, not a backlog or project-memory database. Do not keep unrelated completed or abandoned tasks in the active state merely to preserve history.

## 4. What belongs in state.json

Persist only information that a fresh agent session would need in order to continue correctly.

### `objective`

A concise statement of the current task.

### `acceptance_criteria`

Observable conditions that define completion. Keep only criteria relevant to the current task.

### `constraints`

User requirements, compatibility requirements, architectural constraints, or explicit non-goals that can affect later decisions.

### `facts`

Verified, durable findings that are not obvious enough to rediscover on every step.

Good examples:

- `Authentication is enforced in middleware/auth.go, not in individual handlers.`
- `The public API must remain backwards compatible.`

Bad examples:

- `I ran grep.`
- `I first thought the bug was in handler.go.`

### `decisions`

Decisions already made that later steps should respect, with enough rationale to prevent needless re-litigation when the rationale matters.

### `hypotheses`

Only hypotheses that are still operationally useful. Use objects with this shape:

```json
{
  "statement": "Version increment occurs outside the critical section",
  "status": "open"
}
```

Allowed hypothesis statuses: `open`, `confirmed`, `rejected`.

Remove rejected hypotheses when they no longer help avoid repeating work.

### `files.relevant`

Files that are currently important to the task.

### `files.modified`

Files changed for the current objective. Keep this aligned with the actual working tree.

### `verification.checks`

Record concise verification results using objects such as:

```json
{
  "name": "go test ./...",
  "result": "passed",
  "details": "All packages passed"
}
```

Allowed `result` values: `passed`, `failed`, `not_run`.

Do not store raw logs when a short result is sufficient.

`verification.overall` must be one of: `not_run`, `partial`, `passed`, `failed`.

### `blockers`

Only unresolved conditions preventing progress. Remove resolved blockers.

### `next_action`

One concrete next execution step. It must be actionable by a fresh session without reconstructing the previous conversation.

## 5. What must never be stored

Do not use `state.json` as a transcript or summary of the conversation.

Never store:

- chain-of-thought or hidden reasoning;
- chronological narration of what the agent did;
- copied chat messages unless a specific user requirement must be preserved;
- raw tool output when a concise fact or verification result is enough;
- speculative detail that has no effect on future steps;
- secrets, credentials, tokens, private keys, or sensitive environment values;
- stale facts that are contradicted by the current repository.

Prefer current facts over historical narrative.

## 6. Update cadence

Do not rewrite `state.json` after every read, grep, or shell command.

Update it at meaningful checkpoints, especially when any of these occur:

- the objective or acceptance criteria become clearer;
- a durable fact is discovered;
- a hypothesis is confirmed or rejected;
- an implementation decision is made;
- relevant or modified files change materially;
- verification succeeds or fails;
- a blocker appears or is resolved;
- `next_action` changes;
- the agent is about to give a final response for the current work block;
- the context may be compacted, reset, or handed to a fresh session.

State maintenance is internal bookkeeping. Do not report routine state reads or writes to the user unless they ask or the state itself reveals a problem that affects the task.

## 7. Fresh-session behavior

A fresh session must be able to continue without the previous chat.

At the beginning of non-trivial repository work:

1. Read this procedure.
2. Read `.skillstate/state.json`.
3. If it contains an active matching objective, inspect the current repository as needed to validate the state.
4. Continue from the current objective and `next_action`.
5. Do not ask the user to repeat information already captured in the state or directly observable in the repository.
6. Do not attempt to reconstruct prior conversation history.

The desired continuity model is:

`procedure + current state + current repository/latest observation -> next work`

not:

`entire previous conversation -> next work`.

## 8. Completion

Set `status` to `completed` only when the objective is actually complete and required verification has been performed or the lack of verification is explicitly represented.

On completion:

- set `phase` to `complete`;
- set `next_action` to `null`;
- remove resolved blockers;
- ensure `files.modified` reflects the current task;
- keep only final facts and decisions that explain the resulting implementation state;
- set `verification.overall` accurately.

Do not erase the completed state immediately. Leave it available so a fresh session can understand what just finished. Reinitialize it automatically when a later non-trivial request starts a different objective.

## 9. State quality checks

Before saving state, verify all of the following:

- `state.json` remains valid JSON with no comments;
- the state describes the present, not the chronology of the session;
- entries are concise and actionable;
- duplicate and stale entries have been removed;
- `next_action` matches the actual current situation;
- the repository, not the state file, wins when they disagree.
