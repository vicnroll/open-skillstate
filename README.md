# OpenSkillState

Execution state for coding agents, kept as a validated structured file instead of as conversational history.

An open, opinionated implementation of the **SKILL.state** pattern from *Scalable Long-Horizon Agent Skills* ([arXiv:2608.26263v3](https://arxiv.org/html/2608.26263v3)) — with its own decisions where the paper's model does not fit how coding agents actually work. `skstate` is its CLI.

> **Status: design complete, not implemented.** The architecture is settled and recorded — see [`docs/arquitectura.md`](./docs/arquitectura.md) and the 18 decision records in [`docs/adr/`](./docs/adr/). The binary does not exist yet, so this README documents the design rather than a working install.

## The idea

An agent working a long task accumulates its own transcript, and every step carries all previous steps into the next prompt. That grows quadratically and drags stale reasoning forward. The alternative is to keep an explicit, bounded **execution state** — Σ — and rebuild each step's prompt from the task, that state, and the current observation.

```text
conversational                          OpenSkillState

step 1  [ P + O₁ ]                        step 1  [ P + Σ₁ + O₁ ]
step 2  [ P + O₁R₁a₁ + O₂ ]               step 2  [ P + Σ₂ + O₂ ]
step 3  [ P + O₁R₁a₁ + O₂R₂a₂ + O₃ ]      step 3  [ P + Σ₃ + O₃ ]

        ^ grows with history                      ^ bounded
```

## What makes it hold

The agent **does not write the state file**. It proposes a patch; a small binary owns the schema, validates it, applies the merge and writes atomically, so a malformed patch cannot corrupt persistent state. That is the rule the skill teaches — how strictly it can be enforced depends on the client, and the table below says so plainly.

```bash
skstate patch --stdin <<'EOF'
{ "facts": { "auth-in-middleware": "Auth is enforced in middleware/auth.go, not in handlers" } }
EOF
```

Patches are [RFC 7386 JSON Merge Patch](https://datatracker.ietf.org/doc/html/rfc7386) with no extensions. Collections are objects keyed by content-derived slugs, so a patch costs proportional to what changed, `null` deletes a single item, and restating a known fact overwrites its key instead of accumulating a near-duplicate.

## Guarantees, stated honestly

| Level | Mechanism | Where |
|---|---|---|
| Prevention | A `deny` permission rule stops the model editing the file by hand | Claude Code only |
| Detection | The CLI keeps an integrity hash outside the file; on a mismatch it validates the state and refuses only if it is malformed | Everywhere |
| Input validation | JSON Schema on the tool input, checked by the client | Where an MCP server is configured |

Enforcement is not portable — permission rules and hooks are client-specific. Detection is, but it catches **corruption rather than shortcuts**: the orchestrator's Σ is versioned, so `git pull` and `git checkout` rewrite it legitimately without going through the CLI, and treating every mismatch as an alarm would train everyone to ignore it. A hand-edit that leaves the file well-formed goes unnoticed; one that does not is refused. The threat model is a model taking a shortcut, not an adversary — and that shortcut usually leaves the file malformed, precisely because nothing validated it on the way in.

## Under orchestration

This is where the pattern actually pays off, because a worker launched headlessly starts cold and Σ is all it receives.

```text
   orchestrator ── consolidated Σ (versioned)
        │
        ├── worktree A ── ephemeral Σ (ignored)  ──┐
        ├── worktree B ── ephemeral Σ (ignored)  ──┤── skstate merge
        └── worktree C ── ephemeral Σ (ignored)  ──┘
```

Each worker writes only its own Σ, so nothing contends. Declaring one is a single step when the worktree is created — `skstate init --worker` — and from then on every command in that directory finds the ephemeral state on its own, with no flag to repeat. The two files need different names because worktrees share the repository's `.gitignore`, and a worker's Σ must never enter Git: otherwise Git would three-way-merge a file that `merge` already knows how to merge semantically.

`merge` **promotes a declared subset** — what survives the task, not what dies with it — and stops with an error when two workers wrote the same key with different content, rather than silently picking one.

## Commands

| Command | |
|---|---|
| `get` | Render Σ as compact JSON, omitting empty fields. `--pretty` indents, `--raw` dumps the file |
| `patch` | Apply an RFC 7386 patch from stdin |
| `check` | Validate the state, report a schema-version mismatch, and flag a file that changed outside the CLI |
| `init` | Install into a repository. `--worker` declares an ephemeral state in a worktree |
| `merge` | Promote a worker's Σ into the orchestrator's |
| `migrate` | Move state between schema versions, with a backup |
| `schema` | Print the embedded schema. `--version N` for an older one |

`skstate history` exists too, but it is deliberately absent from `--help` and from the skill: it reads a local store of everything OpenSkillState has recorded, and the agent doing the work must never reach it. Keeping history is compatible with the pattern because O(T²) is a problem of *context*, not of *storage* — what costs tokens is chronology entering the prompt, not chronology existing on disk.

A state file older than the binary keeps working, with a warning — only `migrate` changes its version, and it always runs on demand. A state file *newer* than the binary is always rejected.

## Limits

**Strict state discipline is not universally good.** Section 7 of the paper names three cases where its premise fails, and debugging, auditing and exploring fall in them: you do not know at step 3 which observation from step 1 will matter. That is what `mode: exploration` exists for, and using it is not a workaround — it is the correct mode for that work.

**Compaction is not an implementation of this.** A lossy summary reintroduces exactly the context poisoning the pattern attacks. It is what happens when the context plane fails.

**Resumability is encouraged, not guaranteed.** The `Stop` hook that checks for a concrete `next_action` is capped at a limited number of consecutive blocks by the client, so it nudges once and yields.

## What `init` puts in your repository

```text
.openskillstate/
├── state.json          # Σ
├── installed.json      # what init added to your config, so uninstall stays clean
├── worker-state.json   # only under orchestration — ephemeral, never versioned
└── .integrity          # hash of the last write the CLI made; never versioned

.claude/skills/skstate/    # Claude Code
.agents/skills/skstate/    # Codex, OpenCode, other agents
├── SKILL.md
└── references/            # loaded on demand
    ├── schema.md
    └── orchestration.md
```

`init` also edits two files you already own, and both edits are reversible:

- **`.claude/settings.json`** — the `deny` rule and the `Stop` and `SessionStart` hooks are **merged** into whatever you already have; nothing is overwritten. Since JSON has no comments to mark an inserted block, `init` records what it added in `installed.json`, which is why that file is versioned rather than ignored: re-running `init` replaces instead of duplicating, and uninstalling does not have to guess.
- **`CLAUDE.md` and `AGENTS.md`** — a section between `<!-- BEGIN skstate -->` and `<!-- END skstate -->`. Anything you write outside those markers is never touched. With a TTY `init` asks first; without one it requires `--write-instructions` or `--no-write-instructions` and fails if given neither, because a headless install that decides silently is worse than one that stops and says which flag is missing.

The schema is not installed anywhere. The binary embeds every version it knows and applies the one matching each project's `schema_version`, so a project cannot drift from — or quietly relax — the contract the CLI validates against. `skstate schema` prints it.
