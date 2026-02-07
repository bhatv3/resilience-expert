# Resilience Expert — How to Use

This plugin helps analyze and improve service resiliency in a **composable, evidence-based way**.
It is designed to be run **incrementally**, not all at once.

---
## Prerequisites
•	Claude Code CLI installed
•	Target repository checked out locally
•	resilience-expert plugin checked out locally
•	Claude started from the repo you want to analyze


## Running the Resiliency Plugin

From the root of the service repository you want to analyze:
```
cd <service-repo>
export CLAUDE_CODE_MAX_OUTPUT_TOKENS=16384
claude --plugin-dir ../resilience-expert
```

Claude will analyze the current working directory as the target repository, using the resiliency skills and commands provided by the resilience-expert plugin.

Once Claude starts, run:
```
/resilience-map
/resilience-analyze –lens=control-plane | –lens=dependency-isolation | –lens=domain-decoupling | –lens=modulith
```


---

## Core workflow

Use the plugin as a pipeline:

1. Discover what exists
2. Analyze specific resiliency risks
3. Turn findings into execution work
4. Re-run after changes

Each step produces artifacts that can be reviewed independently.

---

## Step 1: Discovery

Run: /resilience-map

This produces:
- `resilience/discovery.json` — factual inventory (entrypoints, control-plane ops, dependencies)
- `resilience/arch-before.mmd` — current-state diagram (facts only)

Use discovery to:
- confirm entrypoints and dependencies
- identify unknowns early
- establish the “before” view

Do not proceed if discovery is clearly incomplete.

---

## Step 2: Analysis

Run: /resilience-analyze

This consumes `discovery.json` and applies resiliency lenses.

Outputs include:
- `resilience/role_assessment.md`
- `resilience/backlog.csv`
- `resilience/dependency_inventory.csv`
- optional `resilience/arch-after.mmd` (Phase-1 internal boundaries only)

---

## Running a single lens (recommended)

You can run analysis **one lens at a time**: /resilience-analyze –lens=

Supported lenses:
- `control-plane`
- `dependency-isolation`
- `domain-decoupling`
- `modulith`

Single-lens runs:
- reduce noise
- are faster to review
- produce partial but valid outputs
- can be combined later

This is the preferred way to run the tool.

---

## Suggested order (practical)

For most services:
1. `control-plane`
2. `dependency-isolation`
3. `domain-decoupling`
4. `modulith` (only if needed)

You do not need to run all lenses.

---

## Turning output into work

Use `resilience/backlog.csv`:
- Each row maps to one ticket
- P0 / P1 / P2 already assigned
- Evidence and validation plans included

Typical approach:
- One Epic per role
- Stories for all P0 and top P1 items

---

## Re-running

Re-run analysis after:
- adding caches or last-known-good reads
- adding timeouts / circuit breakers / bulkheads
- separating control and operational paths

Re-run **only the relevant lens**.

---

## Key rules

- Discovery is factual only
- Analysis requires evidence
- Unknowns must be explicit
- No topology or regionalization assumptions in Phase 1

---

## When Phase 1 is “done” for a role

- Discovery is complete and trusted
- P0/P1 findings are captured as backlog items
- Validation plans exist
- Remaining unknowns are owned