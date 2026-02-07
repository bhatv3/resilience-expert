# Skill: Reporting & Execution Artifacts

## Purpose
Translate discovery and lens findings into clear, reviewable, and executable outputs that teams can prioritize and deliver.

Reporting must be consistent, evidence-backed, Phase 1–scoped, and suitable for planning systems (JIRA, Google Docs, Sheets).

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

---

## Output Structure

All reporting artifacts are written under `resilience/` using a scoped directory model.

### Directory Layout
resilience/
├── discovery/          # Factual system discovery (input to all analysis)
├── lens_* /            # Independent Phase 1 analyses (one per lens)
└── summary/            # Optional aggregation across lenses (only when lens=all)

---

## Primary Outputs

### 1) Role Assessment Document
A human-readable summary for a single Verify role or deployment unit.

Generated:
- once per lens under `resilience/lens_<lens_name>/role_assessment.md`
- once as an aggregate under `resilience/summary/role_assessment.md` (when `lens=all`)

#### Required sections
- Role Overview
- Analyzed Entrypoints & Dependencies
- Key Findings (lens-scoped)
- Proposed Improvements (prioritized)
- Open Questions & Validation Needed

---

### 2) Findings Table (Structured)
Each lens must emit a structured table (CSV-compatible) of findings.

Generated under:
- `resilience/lens_<lens_name>/backlog.csv`
- aggregated under `resilience/summary/backlog.csv` (when `lens=all`)

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

---

### 3) Dependency Inventory (When Applicable)
Generated only when dependencies are relevant to a lens.

Written to:
- `resilience/lens_<lens_name>/dependency_inventory.csv`
- aggregated under `resilience/summary/dependency_inventory.csv` (when `lens=all`)

Each entry MUST include:
- dependency name
- dependency type (internal / third-party / datastore / stream)
- call sites with evidence
- call path classification (best effort)

---

### 4) Backlog-Ready Items
For each **P0–P1 finding**, reporting must generate a backlog-ready description suitable for direct import into planning systems.

Backlog items must include:
- Title
- Context / Problem Statement
- Evidence
- Proposed Change
- Expected Outcome (customer-facing)
- Validation Plan
- Dependencies (if any)

Backlog items must map 1:1 to findings.

---

## Severity Guidelines

- **P0**  
  Direct customer impact or high likelihood of cascading failure under realistic fault conditions.

- **P1**  
  Material risk under partial failure, load, or dependency degradation.

- **P2**  
  Improvement that increases margin of safety or clarity but does not materially change failure behavior.

Severity MUST be justified by evidence and customer impact.

---

## Diagram Outputs

### `arch-before.mmd`
- Source of truth for current architecture
- Reflects discovery facts only
- No inferred behavior or proposed changes
- Generated from `resilience/discovery/`

### `arch-after.mmd` (Optional, Phase 1 only)
Generated only when Phase 1 identifies internal architectural boundary changes, such as:
- control-plane vs operational-plane separation
- durable internal module boundaries
- explicit adapter/facade boundaries

Rules:
- No new deployment units
- No new regions
- No active-active assumptions
- Conceptual boundaries only

---

## Reporting Rules (Non-Negotiable)
- No findings without evidence
- No backlog items without corresponding findings
- Unknowns must be explicitly documented
- Per-lens outputs must stand on their own
- Aggregated outputs must not dilute lens-specific severity

---

## Output Quality Bar
Reporting is acceptable only when:
- Engineers can convert findings directly into work
- Leadership can understand risk reduction at a glance
- Findings are traceable, reviewable, and defensible
- Outputs respect Phase 1 scope and boundaries