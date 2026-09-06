# Under orchestration

Read this when you are about to dispatch isolated or parallel workers, not otherwise. OpenSkillState does not depend on a particular orchestrator or isolation mechanism.

## Two states, not one

An orchestrator or integration workflow isolates each worker — for example with a Git worktree, a repository copy or a container. Each worker keeps its own state; the project keeps a consolidated one. Nobody shares a writable state file, so ordinary worker execution does not contend.

```text
   consolidated ── .openskillstate/state.json          (versioned in Git)
        │
        ├── workspace A ── .openskillstate/worker-state.json   (ephemeral, ignored)
        ├── workspace B ── .openskillstate/worker-state.json
        └── workspace C ── .openskillstate/worker-state.json
```

The two files have different names on purpose. Git worktrees share the repository's `.gitignore`, so one path cannot be ignored in the worker and versioned in the main copy. Other isolation mechanisms use the same semantic split even when they do not use Git.

## Declare the worker once, when you create it

**When you create a worker, run `skstate init --worker` in its isolated workspace.** Once.

```bash
git worktree add ../work-auth -b fix-auth
cd ../work-auth && skstate init --worker
```

The worker inherits the project's `project_id`. From then on every `skstate` command in that directory finds `worker-state.json` and uses it — there is no flag to remember and nothing to repeat.

If you skip this step in a Git worktree, the CLI warns when the directory looks isolated but no worker state exists. Detection is only a safety net; OpenSkillState does not infer the worker role from Git because isolation might be implemented another way.

## Consolidating when the work lands

```bash
skstate merge ../work-auth
```

`merge` **promotes a declared subset, it does not union two states**. What dies with the task — `objective`, `next_action`, `status`, `mode`, local blockers and hypotheses — stays behind. What can outlive it is promoted according to the schema:

- `facts`
- `decisions`
- `constraints`
- `files.modified`
- `verification.checks`

`verification.overall` is not promoted. The runtime derives it again from the checks present in the consolidated state.

When two sources wrote the same semantic key with different content, `merge` **stops and lists the conflicts** rather than silently picking one. Resolve the judgement explicitly with an ordinary patch.

## Git branches use the same rule

The consolidated `state.json` is versioned, but `init` marks it `-merge` in `.gitattributes`. If two Git branches both change it, Git leaves the current branch's version in the worktree and marks the path conflicted instead of performing a textual three-way merge.

The other branch's state can then be promoted explicitly:

```bash
git show MERGE_HEAD:.openskillstate/state.json | skstate merge --stdin
git add .openskillstate/state.json
```

This is the same semantic operation as integrating a worker; Git is only transporting the state, never deciding how two states combine.

## What a worker should know

A worker started in a fresh session should act as if the previous transcript is unavailable. Its working continuity comes from its state plus the repository and any context the client itself loads.

**Write a durable observation down when you make it.** If later work needs it and it is neither in state nor cheaply recoverable from the repository, a fresh worker cannot rely on the previous conversation.

**Leave `next_action` concrete before finishing.** Whoever picks the work up next should be able to act from the state and repository alone.
