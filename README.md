# OpenSkillState

Execution state for coding agents, kept as a validated structured document instead of relying on conversational history.

An open, opinionated implementation of the **SKILL.state** pattern from *Scalable Long-Horizon Agent Skills* ([arXiv:2608.26263v3](https://arxiv.org/html/2608.26263v3)) — with its own decisions where the paper's model does not fit how coding agents actually work. `skstate` is its CLI.

> **Status: v1 design complete, not implemented.** The architecture is settled in 20 ADRs, but the binary does not exist yet — so this README documents the design rather than a working install.

## The idea

A coding agent doing long-running work can accumulate a transcript and repeatedly carry old reasoning into later prompts. OpenSkillState instead keeps an explicit current execution state — Σ — and makes that state the durable continuity mechanism.

```text
conversational                          state-based execution

step 1  [ P + O₁ ]                        step 1  [ P + Σ₁ + O₁ ]
step 2  [ P + O₁R₁a₁ + O₂ ]               step 2  [ P + Σ₂ + O₂ ]
step 3  [ P + O₁R₁a₁ + O₂R₂a₂ + O₃ ]      step 3  [ P + Σ₃ + O₃ ]

        ^ depends on trajectory                    ^ does not require prior transcript
```

OpenSkillState fully owns the **state plane**: schema, validation, merge semantics, canonical persistence and promotion. The **context plane** still belongs to the client or runtime. A fresh worker can avoid depending on a previous transcript, but the client may still load its own instructions, memory or other context.

For example, Claude Code Auto Memory is separate from OpenSkillState and may be loaded alongside Σ. `init` does not disable it: OpenSkillState does not claim exclusive ownership of the whole prompt.

## What makes the state hold

The agent **does not write the state file directly**. It proposes an RFC 7386 JSON Merge Patch; the runtime normalizes keys, applies the patch, canonicalizes the document, recalculates derived fields, validates the result and writes atomically.

```bash
skstate patch --stdin <<'EOF'
{ "facts": { "auth-in-middleware": "Auth is enforced in middleware/auth.go, not in handlers" } }
EOF
```

Collections are objects keyed by content-derived slugs rather than arrays. A slug is kebab-case, **1 to 4 words**:

```text
readme
auth-in-middleware
public-api-stays-compatible
```

The CLI normalizes mechanical spelling differences such as case, underscores, accents and duplicate hyphens. It never invents semantic truncation when a key exceeds four words.

## Canonical state

RFC 7386 uses `null` to delete a property. That is useful for collection entries, but core state needs one canonical physical shape. After applying the merge patch, the CLI restores missing core fields to their canonical empty representation before validation:

```text
objective, next_action     → null
facts, decisions           → {}
files                      → {relevant:{}, modified:{}}
verification               → {checks:{}, overall:"not_run"}
```

Required runtime fields such as `status`, `mode`, `schema_version` and `project_id` cannot be deleted.

`skstate get` is a renderer, not `cat`:

| Tier | Fields | Rendered by `get` |
|---|---|---|
| core | status, mode, objective, next_action, facts, decisions, files, verification | always, even empty |
| extended | hypotheses, blockers, constraints | only with content |
| meta | schema_version, project_id, project | never |

`get` emits compact one-line JSON; `--pretty` indents the view and `--raw` returns the persisted document.

## Stable project identity

Every initialized project gets an immutable UUID v4:

```json
{
  "project_id": "550e8400-e29b-41d4-a716-446655440000",
  "project": "open-skillstate"
}
```

`project_id` is generated once by `skstate init`, inherited by isolated workers and used to key the local history store. `project` is only a human-readable mutable name. Moving or renaming a repository therefore does not split its historical series, and two repositories with the same name do not collide.

Because the identity must be unique, `init` builds the initial state itself; there is no static `state.json` template in the distributable payload.

## Verification is derived correctly

Agents write individual checks. The aggregate is runtime-owned:

```text
no checks / all not_run → not_run
any failed              → failed
all passed              → passed
passed + not_run        → partial
```

`verification.checks` can be promoted from isolated work. `verification.overall` cannot: it is recomputed from the checks present in the destination. A patch that tries to write `overall` directly is rejected.

## Guarantees, stated narrowly

| Level | Mechanism | Where |
|---|---|---|
| Prevention | Client permission rule blocks direct edits | Where supported; Claude Code in v1 |
| Detection / validation | Integrity hash plus schema validation after external changes | Portable |
| Input validation | Typed tool input / JSON Schema | Where an MCP or equivalent interface is configured |

Claude Code gets this project rule:

```json
{
  "permissions": {
    "deny": ["Edit(/.openskillstate/**)"]
  }
}
```

The path is anchored to the primary working directory of project settings. `Edit(path)` is the relevant Claude Code permission family for built-in file writes; a separate `Write(path)` rule is not installed.

The portable integrity mechanism detects **corruption, not provenance**. Git operations legitimately rewrite the versioned state without going through the CLI. If the hash changes, OpenSkillState validates the document: a well-formed external change is reported and accepted as the new baseline; a malformed one is refused.

A hand edit that remains schema-valid is not distinguishable from every other valid external rewrite. The threat model is an agent taking a shortcut, not an adversary.

## Isolated and parallel work

OpenSkillState is **orchestrator-agnostic**. It does not depend on Syntony, Orca or any other runtime. Worktrees, repository copies and containers are examples of isolation mechanisms, not dependencies.

```text
   consolidated ── .openskillstate/state.json
        │
        ├── workspace A ── worker-state.json  ──┐
        ├── workspace B ── worker-state.json  ──┤── skstate merge
        └── workspace C ── worker-state.json  ──┘
```

Declare an isolated worker once:

```bash
cd ../work-auth
skstate init --worker
```

It inherits the project's `project_id`. From then on commands in that workspace use `worker-state.json` automatically.

`merge` **promotes**, it does not union full states. The v1 promoted set is:

- `facts`
- `decisions`
- `constraints`
- `files.modified`
- `verification.checks`

Task-local fields such as `objective`, `next_action`, `status`, `mode`, blockers and hypotheses stay local. Same-key/different-value conflicts stop the merge and require an explicit decision.

## Git transports the consolidated state, but never merges it

The worker state is ignored. The consolidated state is versioned because it carries durable project continuity, but Git does not know OpenSkillState merge semantics.

`init` therefore adds:

```gitattributes
.openskillstate/state.json -merge
```

If two branches both modify `state.json`, Git keeps the current branch's version provisionally and marks the path conflicted instead of performing a textual three-way merge.

Resolve by promoting the other branch's semantic state:

```bash
git show MERGE_HEAD:.openskillstate/state.json | skstate merge --stdin
git add .openskillstate/state.json
```

This uses the same semantic promotion whether the source came from a human feature branch, a worktree or an external orchestrator.

## State budget

Semantic keys prevent several forms of accidental growth, but they do **not mathematically bound Σ**. A perfectly valid state can still accumulate too much distinct information.

`skstate check` therefore reports state health from v1:

```text
state: valid
state size: 14.2 KiB
facts: 87
  8.6 KiB
decisions: 23
  2.1 KiB
constraints: 12
  1.0 KiB
```

The size is measured on the compact `get` representation — what would actually enter context — rather than the pretty file on disk.

There is **no arbitrary universal cap** such as 8 KiB, 16 KiB or 100 entries. A user or CI can supply a soft budget:

```bash
skstate check --budget-bytes 16384
```

Exceeding it produces a warning, not invalid state and not automatic pruning. The runtime can measure state; deciding which knowledge is no longer necessary remains a semantic decision.

The O(T) argument therefore has an explicit condition: it holds when Σ remains bounded. V1 makes residual growth visible instead of pretending the schema guarantees that condition.

## Commands

| Command | |
|---|---|
| `get` | Render Σ as compact JSON according to tiers. `--pretty` indents, `--raw` returns the document |
| `patch` | Apply RFC 7386 from stdin, canonicalize, derive and validate |
| `check` | Validate state, schema version and integrity; report state size/composition. `--budget-bytes N` adds a soft budget |
| `init` | Initialize a project and install integrations. `--worker` declares an isolated state |
| `uninstall` | Remove client integrations while preserving workspace, state and project identity |
| `deinit` | Remove the OpenSkillState project/workspace. History is only purged with an explicit flag |
| `merge` | Promote semantic state from another workspace/file; `--stdin` accepts a complete source state |
| `migrate` | Move state between schema versions, with a backup |
| `schema` | Print the embedded schema. `--version N` selects an older one |

`skstate history` exists as a separate analysis surface but is deliberately absent from the normal skill and help flow. `skstate hook ...` is an internal client entry point.

## History is stored, not injected

OpenSkillState records metrics and historical events in a local SQLite store outside the repository. Keeping chronology is compatible with the pattern because O(T²) is a problem of **context**, not storage.

The working agent's normal flow does not receive that history: `get`, hooks and the work skill never inject or summarize it. This is **non-exposure, not a security boundary**. An agent with general shell access could theoretically discover a hidden subcommand; the product simply does not advertise or automatically use it in the execution workflow.

History is keyed by `project_id`, not path or project name. It can contain project content and survive repository deletion, so explicit retention and `history purge` are part of the design.

## `init`, `uninstall`, and `deinit`

`init` constructs the canonical initial state and unique identity, installs skills, `.gitignore` and `.gitattributes` entries, Claude Code permissions/hooks where applicable, and optional instruction blocks in `CLAUDE.md` / `AGENTS.md`.

Changes to shared user-owned files are merged rather than overwritten. `.openskillstate/installed.json` records exactly what OpenSkillState inserted so later operations do not have to guess ownership.

`uninstall` removes the agent/client integration — skills, hooks and instruction blocks — **without deleting state or identity**. The repository remains an initialized OpenSkillState project and can keep using the CLI or reinstall integrations later.

`deinit` is the full project teardown. It removes integration, OpenSkillState Git policy and the workspace. It requires explicit confirmation because it deletes Σ. It does not silently delete the external history; `--purge-history` must be requested if that is also desired.

## Schema versions

The schema lives inside the binary, not in each repository. A project cannot quietly relax the contract locally.

Migration is explicit:

- a state newer than the binary is always rejected;
- an older supported state can continue operating with a warning;
- `migrate` changes versions and creates a backup;
- migrations compose consecutive steps (`1→2→3`), so old migration paths remain available indefinitely;
- `project_id` survives migration unchanged.

## Limits

**Strict state discipline is not universally good.** Debugging, auditing and exploratory work can depend on observations whose relevance only becomes clear later. `mode: exploration` relaxes pressure to keep a concrete `next_action` or prematurely structure uncertain findings.

**Compaction is not the context plane.** Lossy summarization can discard the observation that later turns out to matter.

**OpenSkillState does not own every source of client context.** It provides structured execution continuity and lets fresh sessions avoid depending on a previous transcript; client memory and instructions may still coexist.

**A valid state is not automatically a small state.** State budget instrumentation exists because boundedness is a design goal that must be observed, not assumed.

## Design docs

- [`docs/arquitectura.md`](./docs/arquitectura.md) — the complete architecture
- [`docs/adr/`](./docs/adr/) — the 20 accepted decisions and their rationale
- [`CONTEXT.md`](./CONTEXT.md) — project vocabulary and distinctions
- [`schema/v1.json`](./schema/v1.json) — the v1 state contract embedded by the future binary
