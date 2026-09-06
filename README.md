# SKILL.state repository kit

Drop-in template for using the SKILL.state execution-state pattern with interactive Codex, Claude Code, and OpenCode sessions.

## What is included

```text
.
├── .skillstate/
│   ├── procedure.md
│   └── state.json
├── .agents/skills/skill-state/SKILL.md    # Codex
├── .claude/skills/skill-state/SKILL.md   # Claude Code
├── .opencode/skills/skill-state/SKILL.md # OpenCode
├── AGENTS.md
└── CLAUDE.md
```

The three `SKILL.md` files intentionally contain the same portable skill. They are duplicated only so each client finds it in its native project-level location.

## Integrating into an existing repository

1. Copy `.skillstate/` into the repository root.
2. Copy the skill directory for each agent you use, or keep all three.
3. **Do not replace an existing `AGENTS.md` or `CLAUDE.md`.** Merge only the `SKILL.state execution continuity` section into the project's existing instruction file.
4. Keep the rest of the project's normal architecture, build, test, style, security, and review instructions unchanged.

Once installed, the user should work normally. They do **not** need to say "use SKILL.state", "read state.json", or similar wording. The durable project instructions tell the agent when to activate the skill autonomously.

## How the workflow behaves

For non-trivial work the agent should automatically:

1. load the SKILL.state procedure and current state;
2. initialize or resume the current objective;
3. work normally in the repository;
4. update state at meaningful checkpoints;
5. leave a concrete `next_action` when work remains;
6. resume from that state when a later session starts with fresh conversational context.

`state.json` is execution continuity, not project memory and not a transcript.

## Important limitation of interactive clients

The skill can maintain explicit state autonomously, but a repository skill cannot force Codex, Claude Code, or OpenCode to erase the current chat transcript. The strongest approximation to the paper is obtained when a fresh session/context is periodically used for long-running work. The package is designed so that, after such a reset, the agent can resume from `.skillstate/state.json` without the user re-explaining the task.

Do not use conversation compaction as a substitute for state discipline: information needed after a reset should already be represented in the repository or in `state.json`.

## Mutable state and Git

`.skillstate/state.json` is intentionally mutable execution state. Decide per project whether it should be committed, ignored, or handled with another local policy. This kit does not impose a Git policy because team workflows differ.

## Reference

Pattern based on *SKILL.state: Scalable Long-Horizon Agent Skills* (arXiv:2608.26263v2):

https://arxiv.org/html/2608.26263v2
