---
name: skstate
description: Maintain explicit execution state for non-trivial repository work that may span multiple tool calls, edits, debugging or test cycles, user turns, context compaction, or sessions. Invoke autonomously whenever durable execution continuity would help; do not wait for the user to mention it. Skip simple one-shot Q&A and tiny atomic tasks that need no continuity.
compatibility: Claude Code, other AI coding agents
---

# skstate

Keep execution state in a validated structured file instead of in conversational history, so work continues correctly across turns, compaction and fresh sessions.

**You never write the state file directly.** The `skstate` binary owns the schema, validates every change and writes atomically. You read state with `skstate get` and change it with `skstate patch`. Editing the file by hand is blocked where the client supports it, and detected everywhere else.

## When to use this

Invoke this yourself when work is non-trivial and continuity is useful: investigation followed by implementation, multi-file changes, debugging or repeated test/fix cycles, refactors and migrations, or anything likely to span several turns or sessions.

Do not use it for simple explanatory Q&A or a tiny atomic task you can finish immediately. Do not ask the user to name this skill or remind you to read state.

## Starting

Before substantial work:

```bash
skstate get
```

It returns compact single-line JSON — the state with empty fields omitted, so an absent field means empty, not missing. It is a rendered view, not the file; `--pretty` indents it and `--raw` dumps the file itself.

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

Keys absent from the patch are untouched. `null` deletes:

```bash
skstate patch --stdin <<'EOF'
{ "blockers": { "waiting-on-api-key": null } }
EOF
```

### Collection keys are content slugs

Collections (`facts`, `decisions`, `blockers`, `constraints`, `files.*`) are objects, not arrays. You choose each key by compressing the content into a slug:

- kebab-case, 2 to 4 words, no accents, `^[a-z0-9]+(-[a-z0-9]+){0,3}$`
- derive it from the item itself — `auth-in-middleware`, `api-must-stay-compatible`
- reuse the same slug when restating the same item, so the write is idempotent instead of creating a near-duplicate

The CLI **normalises typography for you**: it lowercases, turns `_` into `-`, strips accents and collapses stray hyphens, so `Staging_DB_Unreachable` is stored as `staging-db-unreachable`. Write it properly anyway — you will read the canonical form back from `get`, and matching it is what keeps rewrites idempotent.

What it will **not** fix, because fixing it would mean deciding for you: a slug longer than four words. `handle-rate-limit-errors-gracefully` is rejected, not truncated — shorten it yourself to `rate-limit-handling`.

Two keys in one patch that normalise to the same slug are rejected together, rather than one silently overwriting the other. Patches apply whole or not at all.

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
| `verification` | `checks` and `overall` |

Three more fields exist for cases that need them — `hypotheses`, `blockers`, `constraints`. They are documented in [`references/schema.md`](./references/schema.md); read it when one of them applies. They do not appear in `skstate get` output until they have content, so an absent field means empty, not missing.

## Execution mode and exploration mode

Set `mode` to `exploration` when debugging, auditing or exploring — when you do not yet know which observations will matter, and forcing them into structured state would distort the work. In this mode nothing requires you to keep a concrete `next_action`.

Switch back to `execution` when the objective becomes concrete. Changing mode is a patch like any other:

```bash
skstate patch --stdin <<'EOF'
{ "mode": "execution", "next_action": "Fix the nil check in parser.go:88" }
EOF
```

## What belongs in state, and what does not

Keep: verified facts needed later, constraints from the user, decisions future steps must respect, concise verification outcomes, unresolved blockers, and one concrete `next_action`.

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

## Orchestrating parallel workers

If you are dispatching workers rather than doing the work yourself — worktrees, parallel tasks, an integration step afterwards — read [`references/orchestration.md`](./references/orchestration.md) before you start. There is one thing to do when you create each worker, and skipping it corrupts the shared state silently.

## Fresh sessions

A fresh session continues from `skstate get` plus the current repository. Do not try to reconstruct the previous transcript, and do not ask the user to restate what is already in state or visible in the repository.
