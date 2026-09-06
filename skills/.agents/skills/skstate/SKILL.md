---
name: skstate
description: Maintain explicit execution state for non-trivial repository work that may span multiple tool calls, edits, debugging or test cycles, user turns, context compaction, or sessions. Invoke autonomously whenever durable execution continuity would help; do not wait for the user to mention it. Skip simple one-shot Q&A and tiny atomic tasks that need no continuity.
compatibility: Claude Code, other AI coding agents
---

# skstate

Keep execution state in a validated structured file instead of relying on conversational history, so work can continue correctly across turns, compaction and fresh sessions.

**You never write the state file directly.** The `skstate` binary owns the schema, validates every change and writes atomically. You read state with `skstate get` and change it with `skstate patch`. Editing the file by hand is blocked where the client supports it; elsewhere the CLI detects an external change on the next operation and refuses it if the resulting state is malformed.

## When to use this

Invoke this yourself when work is non-trivial and continuity is useful: investigation followed by implementation, multi-file changes, debugging or repeated test/fix cycles, refactors and migrations, or anything likely to span several turns or sessions.

Do not use it for simple explanatory Q&A or a tiny atomic task you can finish immediately. Do not ask the user to name this skill or remind you to read state.

## Starting

Before substantial work:

```bash
skstate get
```

It returns compact single-line JSON. **Core fields are always present**, even when empty; extended fields are omitted until they contain something; meta fields such as `schema_version`, `project_id` and `project` are not emitted. It is a rendered view, not the file; `--pretty` indents it and `--raw` dumps the persisted document.

Compare the user's latest request against what comes back. If it is the same objective, reconcile with the current repository and continue from `next_action`. If it is a different objective, or state is `idle`/`completed`, reinitialize for the new task. The user's latest instruction and the current repository always win over stale state.

## Changing state

Send a JSON Merge Patch (RFC 7386) on stdin. Only what changes:

```bash
skstate patch --stdin <<'EOF'
{
  "facts": { "auth-in-middleware": "Auth is enforced in middleware/auth.go, not in handlers" },
  "next_action": "Add the missing timeout test to auth_test.go"
}
EOF
```

Keys absent from the patch are untouched. `null` deletes a collection item. For core fields such as `next_action`, the runtime canonicalizes the document after the merge, so `{"next_action": null}` leaves the canonical empty field present in `state.json`.

```bash
skstate patch --stdin <<'EOF'
{ "blockers": { "waiting-on-api-key": null } }
EOF
```

### Collection keys are content slugs

Collections (`facts`, `decisions`, `blockers`, `constraints`, `files.*`) are objects, not arrays. You choose each key by compressing the content into a slug:

- kebab-case, **1 to 4 words**, no accents, `^[a-z0-9]+(-[a-z0-9]+){0,3}$`
- derive it from the item itself — `readme`, `auth-in-middleware`, `api-stays-compatible`
- reuse the same slug when restating the same item, so the write is idempotent instead of creating a near-duplicate

The CLI **normalizes typography for you**: it lowercases, turns `_` into `-`, strips accents and collapses stray hyphens, so `Staging_DB_Unreachable` is stored as `staging-db-unreachable`. Write the canonical form when practical because matching it is what keeps rewrites idempotent.

What it will **not** fix, because fixing it would mean deciding for you: a slug longer than four words. `handle-rate-limit-errors-gracefully` is rejected, not truncated — shorten it yourself to `rate-limit-handling`.

Two keys in one patch that normalize to the same slug are rejected together rather than one silently overwriting the other. Patches apply whole or not at all.

## The state shape

| Field | Meaning |
|---|---|
| `status` | `idle` \| `active` \| `blocked` \| `completed` |
| `mode` | `execution` \| `exploration` |
| `objective` | What this task is, in one sentence |
| `next_action` | One concrete next step, actionable without the prior conversation |
| `facts` | Verified durable findings not worth rediscovering each step |
| `decisions` | Decisions later steps must respect, with rationale where it matters |
| `files` | `relevant` and `modified` |
| `verification` | `checks`; `overall` is derived by the runtime |

Three more fields exist for cases that need them — `hypotheses`, `blockers`, `constraints`. They are documented in [`references/schema.md`](./references/schema.md); read it when one of them applies. They do not appear in `skstate get` output until they have content.

`project_id` and `schema_version` are runtime-owned meta fields. Do not try to patch them. `project` is a mutable display name but normally does not need to enter execution-state maintenance.

## Execution mode and exploration mode

Set `mode` to `exploration` when debugging, auditing or exploring — when you do not yet know which observations will matter, and forcing them into structured state would distort the work. In this mode nothing requires you to keep a concrete `next_action`.

Switch back to `execution` when the objective becomes concrete. Changing mode is a patch like any other:

```bash
skstate patch --stdin <<'EOF'
{ "mode": "execution", "next_action": "Fix the nil check in parser.go:88" }
EOF
```

## What belongs in state, and what does not

Keep: verified facts needed later, constraints from the user, decisions future steps must respect, concise verification checks, unresolved blockers, and one concrete `next_action`.

Never store: chain-of-thought or private reasoning, narration of what you did in what order, copied conversation history, raw logs where a short result suffices, stale or duplicated entries, secrets or credentials.

The state describes **the present, not the chronology**. If a fact is cheap to recover from the repository, prefer the repository over enlarging the state — unless keeping it prevents repeating real investigation.

## Cadence

Patch at meaningful checkpoints, not after every tool call. A checkpoint is when something a future fresh session would need has changed: the objective, a durable fact, a decision, verification results, blockers appearing or clearing, or the next action.

Always patch before your final response for a work block, and before anything likely to reset context.

Do not report routine state maintenance to the user unless they ask, or unless the state itself reveals a problem affecting the task.

## Finishing

When the objective is genuinely complete and verified:

```bash
skstate patch --stdin <<'EOF'
{ "status": "completed", "next_action": null }
EOF
```

Leave the completed state in place. It gets reinitialized when a different objective starts.

## State growth

`skstate check` reports the size and composition of the rendered state. A large valid state may still be a poor execution state. If a configured budget warning fires, remove stale, duplicated or cheaply recoverable information deliberately; never summarize or delete content automatically just to hit a number.

## Orchestrating parallel workers

If you are dispatching isolated workers rather than doing the work yourself — worktrees, containers, copies or parallel tasks with an integration step afterwards — read [`references/orchestration.md`](./references/orchestration.md) before you start. Declare each worker once when creating its workspace; OpenSkillState does not depend on any particular orchestrator.

## Fresh sessions

A fresh session continues from `skstate get` plus the current repository and whatever additional context the client itself loads. Do not try to reconstruct the previous transcript, and do not ask the user to restate what is already in state or visible in the repository.
