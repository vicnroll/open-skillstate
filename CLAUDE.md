# skill-state-kit

This repository designs and (soon) implements `skillstate`: execution state for coding agents, kept as a validated structured file instead of as conversational history.

## Current phase

**The design is complete; the binary does not exist yet.** There is no Go module, no executable, no tests. Commands like `skillstate get` appear throughout the documentation because they describe the product being built, not something installed here.

## Where things live

| | |
|---|---|
| `docs/arquitectura.md` | What the system is and how it works, as a whole |
| `docs/adr/` | The 15 design decisions and why each was made |
| `CONTEXT.md` | Project vocabulary. Check terms here before inventing new ones |
| `skills/` | The payload `init` will install: skill, initial state, instruction fragment |
| `schema/` | The schema versions the binary embeds. Source, not payload — never installed |
| `.skillstate/state.json` | Working state for the design effort itself |

**Do not re-argue what the ADRs already settle.** If you believe a decision is wrong, say so and cite the ADR; do not silently design around it.

**The old prototype is not normative, and it is not here.** A text-only kit based on v2 of the paper came first. It is deliberately absent from the working tree and survives only as the initial commit `4d7b571`, so nothing you find by searching this repository comes from it. Do not restore it, and do not treat what it promised as a constraint.

## Language

Distributable artefacts are written in **English**: `README.md`, `skills/**` (SKILL.md, references, schema, instruction fragment). Design artefacts are written in **Spanish**: `docs/arquitectura.md`, `docs/adr/**`, `CONTEXT.md`. Keep that split.

## Execution continuity

`.skillstate/state.json` holds the working state of this design effort. Read it before substantial work and keep it current at meaningful checkpoints.

It is maintained **by hand**, as plain design notes, because the tool that would own it is what we are building. That is a deliberate exception, not the pattern the product prescribes — once the binary exists, `init` replaces this section and all writes go through `skillstate patch`.

Keep it as compact current state, never a transcript. No chain-of-thought, no chronology of what was done in what order, no secrets, no raw tool output. If something is cheap to recover from the repository, prefer the repository over enlarging the state.
