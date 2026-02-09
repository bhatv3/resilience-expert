# Resilience Expert — How to Use

This plugin helps analyze and improve service resiliency in a **composable, evidence-based, Phase-1–scoped way**.

It is designed to be run **incrementally**, lens by lens — not as a single “big analysis”.

The output is intended for **engineers and architects to review, prioritize, and execute**, not for automated decision-making.

---

## Core workflow

Use the plugin as a pipeline:

1. Discover what exists (facts only)
2. Analyze specific resiliency risks (by lens)
3. Turn findings into execution work
4. Re-run only the lenses affected by changes

Each step produces artifacts that stand on their own.

---

## Step 1: Discovery

Run: /resilience-map

This performs **factual discovery only**.

### Outputs

Written under: resilience/discovery/

Artifacts:
- `discovery.json`  
  Factual inventory of:
    - operational entrypoints
    - control-plane operations (best effort)
    - outbound dependencies
    - open questions / unknowns

- `arch-before.mmd`  
  Current-state architecture diagram:
    - facts only
    - no inferred topology
    - no proposed changes

### How to use discovery

Use discovery to:
- confirm entrypoints and dependencies
- surface unknowns early
- establish a trusted “before” view

Do **not** proceed to analysis if discovery is clearly incomplete or misleading.

---

## Step 2: Analysis (Phase 1)

Run: /resilience-analyze

This consumes discovery outputs and applies **one or more resiliency lenses**.

Analysis is:
- evidence-backed
- Phase-1–scoped
- topology-agnostic (no regions, no deployable splits)

---

## Lens-scoped execution model (important)

Analysis outputs are **always lens-scoped**.

There is **no aggregated summary output**.

### Directory layout

resilience/
    discovery/
        discovery.json
        arch-before.mmd
    
    lens_plane-separation/
    lens_domain-decoupling/
    lens_modulith/
    lens_dependency-isolation/

Each lens produces **independent, consumable artifacts**.

---

## Running a single lens (recommended)

Run one lens at a time: /resilience-analyze –lens=

Supported lenses:
- `plane-separation`
- `domain-decoupling`
- `modulith`
- `dependency-isolation`

Single-lens runs:
- reduce cognitive load
- are faster to review
- avoid overwriting outputs
- align with incremental execution

This is the **preferred and recommended way** to use the tool.

---

## Per-lens outputs

Written under: resilience/lens_/

Each lens emits:
- `role_assessment.md`  
  Human-readable findings and context

- `backlog.csv`  
  Backlog-ready items:
    - severity (P0/P1/P2)
    - customer impact
    - evidence
    - recommended change or investigation
    - validation notes

Optional (only when justified by Phase 1 analysis):
- `dependency_inventory.csv`
- `arch-after.mmd`  
  Conceptual internal boundaries only  
  (no new regions, no new deployables)

If a lens finds nothing actionable, it still emits a `role_assessment.md` stating “No findings” and listing any open questions.

---

## Suggested lens order (practical)

For most services:

1. **plane-separation**  
   (global vs local, control vs operational stability)
2. **domain-decoupling**  
   (regionalization blockers, external contracts)
3. **dependency-isolation**  
   (timeouts, retries, bulkheads on critical paths)
4. **modulith** *(only if needed)*  
   (identify future fault lines, not elegance refactors)

You do **not** need to run all lenses for every role.

---

## Turning output into work

Use `backlog.csv` from each lens:
- Each row maps to **one ticket**
- Severity is pre-classified
- Evidence and validation steps are included

Typical usage:
- One Epic per role
- Stories for all P0 and top P1 findings
- P2s as opportunistic improvements

---

## Re-running analysis

Re-run **only the affected lens** after changes such as:
- adding caches or last-known-good reads
- introducing timeouts, circuit breakers, or bulkheads
- separating control and operational execution paths
- hardening domain adapters

Discovery usually does **not** need to be re-run unless entrypoints or dependencies change.

---

## Key rules (non-negotiable)

- Discovery is factual only
- Analysis requires explicit evidence
- Unknowns must be surfaced, not hidden
- Phase 1 does **not** assume regionalization or topology changes

---

## When Phase 1 is “done” for a role

Phase 1 is complete when:
- discovery is trusted
- P0/P1 findings are captured as backlog items
- validation plans exist
- remaining unknowns are explicitly owned

At that point, outputs can safely inform:
- Phase 2 topology discussions
- Phase 3 regionalization planning

