# Skill: Reporting & Execution Artifacts

## Purpose
Translate discovery and lens findings into clear, reviewable, and executable outputs that teams can prioritize and deliver.

Reporting must be consistent, evidence-backed, and suitable for planning systems (JIRA, Google Docs, Sheets).

---

## Primary Outputs

### 1) Role Assessment Document
A human-readable summary for a single Verify role.

Sections:
- Role Overview
- Key Entrypoints & Dependencies
- Top Resiliency Risks (by lens)
- Proposed Improvements (prioritized)
- Open Questions & Validation Needed

---

### 2) Findings Table (Structured)
Each finding must include:

| Field | Description |
|-----|------------|
| Role | Deployment unit |
| Lens | Which analytical lens |
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

### 3) Backlog-Ready Items
For each P0–P1 finding, generate a backlog-ready description suitable for JIRA:

- Title
- Context / Problem Statement
- Evidence
- Proposed Change
- Expected Outcome
- Validation Plan
- Dependencies (if any)

---

## Severity Guidelines

- **P0:** Direct customer impact or high likelihood of cascading failure
- **P1:** Material risk under partial failure or load
- **P2:** Improvement that increases margin of safety

Severity must be justified by evidence and impact, not intuition.

---

## Diagram Outputs

### arch-before.mmd
- Reflects discovery facts only
- No proposed changes
- Shows:
    - role boundary
    - entrypoints
    - control-plane ops
    - dependencies

### arch-after.mmd (Optional, Phase 1)
- Only if changes are internal and structural (e.g., plane separation)
- Must clearly label conceptual boundaries (not deployment changes)

---

## Reporting Rules
- No findings without evidence
- No backlog items without a corresponding finding
- Unknowns must be explicitly documented
- Improvements must be scoped to Phase 1 unless explicitly stated otherwise

---

## Output Quality Bar
Reporting is successful when:
- Teams can convert outputs directly into work
- Leadership can understand risk reduction at a glance
- Findings are traceable, reviewable, and defensible