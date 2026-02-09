# Skill: Reporting & Execution Artifacts

## Purpose
Translate discovery and lens findings into clear, reviewable, and executable outputs that teams can prioritize and deliver.

Reporting is:
- evidence-backed
- Phase 1–scoped
- lens-specific
- suitable for planning systems (JIRA, Google Docs, Sheets)

---

## Reporting Scope
Reporting consumes outputs from:
- `skills/discovery.md`
- `skills/lenses.md`
- `skills/verification.md`

Reporting does **not**:
- introduce new findings
- infer architecture or topology
- reclassify severity without evidence
- synthesize or summarize across lenses

---

## Output Structure

All reporting artifacts are written under `resilience/` using **lens-scoped directories only**.

There is **no aggregated summary output**.

### Directory Layout

resilience/
    ├── discovery/                  # Factual system discovery (input to all analysis)
    ├── lens_plane-separation/
    ├── lens_domain-decoupling/
    ├── lens_modulith/
    └── lens_dependency-isolation/

Each lens directory is self-contained and independently consumable.

---

## Primary Outputs (Per Lens)

Each lens MUST emit the following artifacts under its own directory:
`resilience/lens_<lens_name>/`

---

### 1) Role Assessment Document

A human-readable assessment scoped to:
- one role / deployment unit
- one analytical lens

Written to:
- `resilience/lens_<lens_name>/role_assessment.md`

#### Required sections
- Role Overview
- Analyzed Entrypoints & Dependencies
- Key Findings (lens-specific)
- Proposed Improvements (prioritized)
- Open Questions & Validation Needed

Notes:
- This document must stand on its own.
- It must not assume findings or context from other lenses.

---

### 2) Findings Table (Structured)

Each lens must emit a CSV-compatible findings table.

Written to:
- `resilience/lens_<lens_name>/backlog.csv`

Each row MUST include:

| Field | Description |
|-----|------------|
| Role | Deployment unit |
| Lens | Analytical lens |
| Finding | Clear description of risk |
| Evidence | File + line range |
| Customer Impact | How users are affected |
| Severity | P0 / P1 / P2 |
| Confidence | High / Medium / Low |
| Improvement | Actionable change |
| Scope | Role-local / Cross-role |
| Effort | S / M / L |
| Notes | Validation or follow-ups |

This table is intended for:
- Google Sheets
- JIRA ticket creation
- cross-role comparison

---

### 3) Dependency Inventory (When Applicable)

Generated **only when dependencies are relevant to the lens**.

Written to:
- `resilience/lens_<lens_name>/dependency_inventory.csv`

Each entry MUST include:
- dependency name
- dependency type (internal / third-party / datastore / stream)
- call sites with evidence
- call path classification (best effort)

If a lens does not materially analyze dependencies, this file may be omitted.

---

### 4) Backlog-Ready Items

For each **P0–P1 finding**, reporting must produce a backlog-ready description suitable for direct import into planning systems.

Backlog items must include:
- Title
- Context / Problem Statement
- Evidence
- Proposed Change
- Expected Outcome (customer-facing)
- Validation Plan
- Dependencies (if any)

Rules:
- Backlog items must map **1:1** to findings.
- No “combined” or speculative tickets.

---

## Severity Guidelines

- **P0**  
  Direct customer impact or high likelihood of cascading failure under realistic fault conditions.

- **P1**  
  Material risk under partial failure, load, or dependency degradation.

- **P2**  
  Improvement that increases margin of safety or clarity but does not materially change failure behavior.

Severity MUST be justified by:
- evidence
- failure mechanics
- customer impact

---

## Diagram Outputs

### `arch-before.mmd`
- Source of truth for current architecture
- Reflects discovery facts only
- No inferred behavior or proposed changes
- Generated from `resilience/discovery/`

### `arch-after.mmd` (Optional, Per Lens)
Generated **only when a lens identifies meaningful Phase 1 internal boundary changes**, such as:
- control-plane vs operational-plane separation
- durable internal module boundaries
- explicit adapter / facade boundaries

Rules:
- No new deployment units
- No new regions
- No active-active assumptions
- Conceptual boundaries only

If no such changes are identified, `arch-after.mmd` must **not** be generated.

---

## Reporting Rules (Non-Negotiable)
- No findings without evidence
- No backlog items without corresponding findings
- Unknowns must be explicitly documented
- Each lens output must be independently understandable
- Outputs must respect Phase 1 scope and boundaries

---

## Output Quality Bar
Reporting is acceptable only when:
- Engineers can convert findings directly into work
- Leadership can assess risk reduction at a glance
- Findings are traceable, reviewable, and defensible
- Outputs remain lightweight and composable