# Under orchestration

Read this when you are about to dispatch parallel workers, not otherwise. Nothing here applies to ordinary single-session work.

## Two states, not one

An orchestrator isolates each worker — usually in its own Git worktree — and runs several at once. Each worker keeps its own state; the orchestrator keeps a consolidated one. Nobody shares a file, so nothing contends.

```text
   orchestrator ── .openskillstate/state.json          (versioned in Git)
        │
        ├── worktree A ── .openskillstate/worker-state.json   (ephemeral, ignored)
        ├── worktree B ── .openskillstate/worker-state.json
        └── worktree C ── .openskillstate/worker-state.json
```

The two files have different names on purpose. Worktrees share the repository's `.gitignore`, so one path cannot be ignored in the worktree and versioned in the main copy — and a worker's state must never enter Git, or Git would try to three-way-merge a file that `skstate merge` already knows how to merge semantically.

## Declare the worker once, when you create it

**When you create a worker, run `skstate init --worker` in its worktree.** Once.

```bash
git worktree add ../work-auth -b fix-auth
cd ../work-auth && skstate init --worker
```

That is the whole protocol. From then on every `skstate` command in that directory finds `worker-state.json` and uses it — there is no flag to remember and nothing to repeat. If you skip this step the worker writes to the versioned state, which is the one failure this design cannot recover from cleanly, so the CLI warns loudly when a directory looks like a worktree and no worker was declared.

## Consolidating when the work lands

```bash
skstate merge ../work-auth
```

`merge` **promotes a declared subset, it does not union two states**. What dies with the task — `objective`, `next_action`, `status` — stays behind; what outlives it — `facts`, `decisions`, `files.modified`, `verification` — is promoted. Otherwise the orchestrator's state would fill with the objectives of a dozen finished workers and grow without bound exactly where it must stay small.

When two workers wrote the same key with different content, `merge` **stops and lists the conflicts** rather than silently picking one. That collision means both wrote about the same thing and disagree, so resolving it is a judgement call. Read both values, decide, and write the answer as an ordinary patch:

```bash
skstate patch --stdin <<'EOF'
{ "facts": { "auth-in-middleware": "Auth runs in middleware for HTTP, and in the job runner for async work" } }
EOF
```

## What a worker should know

A worker starts cold: its state is genuinely everything it receives, with no transcript behind it. That makes two habits matter more than in an interactive session.

**Write the observation down when you make it.** In a conversation you can look back at step 1 from step 7; a worker cannot. An observation that is not in the state did not happen.

**Leave `next_action` concrete before finishing.** Whoever picks the work up next has your state and the repository, nothing else.
