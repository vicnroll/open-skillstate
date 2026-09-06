---
name: skill-state
description: Maintain explicit execution state for non-trivial repository work that may span multiple tool calls, edits, debugging or test cycles, user turns, context compaction, or sessions. Invoke autonomously whenever durable execution continuity would help; do not wait for the user to mention SKILL.state. Skip simple one-shot Q&A and tiny atomic tasks that need no continuity.
compatibility: Codex, Claude Code, OpenCode
---

# SKILL.state

Use explicit mutable execution state so repository work can continue correctly without depending on prior conversational history.

This skill is an operational adaptation of the SKILL.state pattern for interactive coding agents. The canonical repository procedure is `.skillstate/procedure.md`; the mutable execution state is `.skillstate/state.json`.

## Autonomous activation

Invoke this skill yourself when the task is non-trivial and execution continuity is useful. Typical triggers include:

- investigation followed by implementation;
- multi-file changes;
- debugging or repeated test/fix cycles;
- refactors and migrations;
- work likely to span several user turns;
- work that may cross context compaction or a new agent session.

Do not require the user to name this skill, mention SKILL.state, or remind you to read state.

Do not activate it for simple explanatory Q&A or tiny atomic work that can be completed immediately without continuity.

## Required startup behavior

For an applicable task, before substantial work:

1. Read `.skillstate/procedure.md`.
2. Read `.skillstate/state.json`.
3. Compare the user's latest request with the current state.
4. If the state represents the same active objective, reconcile it with the current repository and continue from `next_action`.
5. If the state is idle/completed or represents a different objective, initialize it for the new task according to the procedure.
6. Never follow stale state over the user's latest instruction or the current repository.

Do this without asking the user for SKILL.state-specific confirmation.

## During work

Work normally with the repository and available tools. Treat the filesystem and current tool observations as the live source of truth.

Update `.skillstate/state.json` at meaningful checkpoints, not after every tool call. A checkpoint exists when information necessary for a future fresh session changes, such as:

- objective or acceptance criteria;
- durable facts;
- decisions;
- useful hypotheses;
- relevant or modified files;
- verification outcomes;
- blockers;
- the next concrete action.

Follow the state shape and lifecycle rules in `.skillstate/procedure.md` exactly.

## State discipline

The state must represent the present execution state, not the chronology of the conversation.

Preserve:

- verified facts needed later;
- user constraints and acceptance criteria;
- implementation decisions future steps must respect;
- concise verification outcomes;
- unresolved blockers;
- one concrete `next_action`.

Do not preserve:

- chain-of-thought or private reasoning;
- narrative such as "first I did X, then I did Y";
- copied conversation history;
- raw logs when a concise result is enough;
- stale or duplicate information;
- secrets or credentials.

If a fact can be cheaply and reliably recovered from the current repository, prefer the repository over bloating the state unless retaining the fact prevents meaningful repeated investigation.

## User follow-ups

On each follow-up message for the same task:

- incorporate new constraints or changed acceptance criteria into state when they affect later work;
- continue the active objective without asking the user to restate prior information already stored or visible in the repository;
- if the user clearly changes to a different task, the new instruction wins and the active task state is reinitialized rather than blindly following the old `next_action`.

## Before finishing a work block

Before the final response for a non-trivial work block:

1. Reconcile `state.json` with what actually changed.
2. Record material verification results.
3. Set an accurate `next_action` if work remains.
4. If complete, set `status` to `completed`, `phase` to `complete`, and `next_action` to `null`.
5. Keep the state concise and valid JSON.

Do not announce routine state maintenance unless the user asks about it or a state inconsistency materially affects the task.

## Fresh sessions and context loss

When working in a fresh chat/session, do not depend on the previous conversation. Reconstruct only the minimum needed execution context from:

1. `.skillstate/procedure.md`;
2. `.skillstate/state.json`;
3. the current repository and fresh observations.

If the client compacts or resets conversation context, ensure durable execution information has been captured in state before relying on that reset.

A fresh session should be able to continue from `objective` + current repository + `next_action` without access to the prior transcript.
