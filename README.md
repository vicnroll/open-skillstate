# skillstate

Execution state for coding agents, kept as a validated structured file instead of as conversational history.

Based on *SKILL.state: Scalable Long-Horizon Agent Skills* ([arXiv:2608.26263v3](https://arxiv.org/html/2608.26263v3)).

> **Status: design complete, not implemented.** The architecture is settled and recorded — see [`docs/arquitectura.md`](./docs/arquitectura.md) and the 14 decision records in [`docs/adr/`](./docs/adr/). The binary does not exist yet, so this README documents the design rather than a working install.

## The idea

An agent working a long task accumulates its own transcript, and every step carries all previous steps into the next prompt. That grows quadratically and drags stale reasoning forward. The alternative is to keep an explicit, bounded **execution state** — Σ — and rebuild each step's prompt from the task, that state, and the current observation.

```text
conversational                            skillstate

step 1  [ P + O₁ ]                        step 1  [ P + Σ₁ + O₁ ]
step 2  [ P + O₁R₁a₁ + O₂ ]               step 2  [ P + Σ₂ + O₂ ]
step 3  [ P + O₁R₁a₁ + O₂R₂a₂ + O₃ ]      step 3  [ P + Σ₃ + O₃ ]

        ^ grows with history                      ^ bounded
```

## What makes it hold

The agent **never writes the state file**. A small binary owns the schema, validates every change, applies the merge and writes atomically — so a malformed output cannot corrupt persistent state.

```bash
skillstate patch --stdin <<'EOF'
{ "facts": { "auth-in-middleware": "Auth is enforced in middleware/auth.go, not in handlers" } }
EOF
```

Patches are [RFC 7386 JSON Merge Patch](https://datatracker.ietf.org/doc/html/rfc7386) with no extensions. Collections are objects keyed by content-derived slugs, so a patch costs proportional to what changed, `null` deletes a single item, and restating a known fact overwrites its key instead of accumulating a near-duplicate.

## Guarantees, stated honestly

| Level | Mechanism | Where |
|---|---|---|
| Prevention | A `deny` permission rule stops the model editing the file by hand | Claude Code only |
| Detection | The CLI keeps an integrity hash outside the file and compares on every operation | Everywhere |
| Input validation | JSON Schema on the tool input, checked by the client | Where an MCP server is configured |

Enforcement is not portable — permission rules and hooks are client-specific. Detection is, so a direct write is always at least *loud*, even where it cannot be blocked. The threat model is a model taking a shortcut, not an adversary.

## Under orchestration

This is where the pattern actually pays off, because a worker launched headlessly starts cold and Σ is all it receives.

```text
   orchestrator ── consolidated Σ (versioned)
        │
        ├── worktree A ── ephemeral Σ (ignored)  ──┐
        ├── worktree B ── ephemeral Σ (ignored)  ──┤── skillstate merge
        └── worktree C ── ephemeral Σ (ignored)  ──┘
```

Each worker writes only its own Σ, so nothing contends. `merge` **promotes a declared subset** — what survives the task, not what dies with it — and stops with an error when two workers wrote the same key with different content, rather than silently picking one.

## Commands

| Command | |
|---|---|
| `get` | Render Σ, omitting empty fields. `--raw` dumps the file |
| `patch` | Apply an RFC 7386 patch from stdin |
| `check` | Validate, detect schema drift and direct writes |
| `init` | Install into a repository |
| `merge` | Promote a worker's Σ into the orchestrator's |
| `migrate` | Move state between schema versions, with a backup |

## Limits

**Strict state discipline is not universally good.** Section 7 of the paper names three cases where its premise fails, and debugging, auditing and exploring fall in them: you do not know at step 3 which observation from step 1 will matter. That is what `mode: exploration` exists for, and using it is not a workaround — it is the correct mode for that work.

**Compaction is not an implementation of this.** A lossy summary reintroduces exactly the context poisoning the pattern attacks. It is what happens when the context plane fails.

**Resumability is encouraged, not guaranteed.** The `Stop` hook that checks for a concrete `next_action` is capped at a limited number of consecutive blocks by the client, so it nudges once and yields.

## Layout

```text
.skillstate/
├── schema.json      # the schema the CLI owns
└── state.json       # Σ

.claude/skills/skill-state/    # Claude Code
.agents/skills/skill-state/    # Codex, OpenCode, other agents
├── SKILL.md
└── references/schema.md       # loaded on demand
```
