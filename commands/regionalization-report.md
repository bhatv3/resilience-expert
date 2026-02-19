# Command: /regionalization-report

## Intent

Generate a structured, implementation-grade regionalization decision inventory
by synthesizing outputs from existing resiliency artifacts.

This command:

- DOES NOT re-analyze source code
- DOES NOT apply new lenses
- DOES NOT introduce new findings
- DOES NOT infer topology or regional design
- DOES NOT propose deployment models

It reorganizes existing findings into a structured answer to:

> "What blocks or enables regionalization for this repo,
> and what exactly must we implement to fix it?"

This command is a synthesis layer, not a new analytical lens.

---

## Inputs

Required:
- `resilience/discovery/discovery.json`

Optional (best effort if present):
- `resilience/lens_plane-separation/backlog.csv`
- `resilience/lens_domain-decoupling/backlog.csv`
- `resilience/lens_modulith/backlog.csv`
- `resilience/lens_dependency-isolation/backlog.csv`

If a lens output is missing:
- Proceed with available inputs
- Mark coverage gaps as `unknown`
- Emit a validation note in `open_questions.md`

---

## Scope

This command operates **repo-locally only**.

It produces:
- A dependency-first regionalization inventory
- An implementation-ready change summary per dependency
- Explicit blockers and enablers
- Traceability back to original findings

It does not aggregate across repositories.

---

## Execution Steps (In Order)

### 1) Load Discovery

Parse:
- role metadata
- entrypoints
- dependencies
- call paths
- open questions

Build internal maps:
- dependency map
- entrypoint map
- call-path classification

No inference beyond discovery artifacts.

---

### 2) Load Lens Findings

For each lens backlog file present:

Normalize each finding into:

- lens
- finding description
- severity
- confidence
- evidence
- improvement
- effort
- notes

Preserve traceability to original lens outputs.

If backlog file is malformed or missing:
- Mark lens coverage as `unknown`
- Add note to `open_questions.md`

---

### 3) Map Findings to Dependencies

For each dependency in discovery:

Attach findings that:

- Explicitly reference dependency name
- Reference known datastore identifiers
- Reference integration adapters or call sites
- Reference related SDK or client usage

If no findings map to a dependency:
- Regionalization posture remains `unknown`
- Emit note in open_questions

No heuristic inference beyond explicit evidence.

---

### 4) Derive Regionalization Posture (Deterministic)

Fields must be derived strictly from mapped findings.

#### runtime_criticality
From Lens 4 classification only.
Else `unknown`.

Values:
- hard
- soft
- non
- unknown

---

#### global_ok / must_be_local
From Lens 1 findings only.

Rules:
- Explicit region-local requirement → must_be_local=yes
- Global allowed with cache/LKG → global_ok=conditional
- Explicit safe-global → global_ok=yes
- Else → unknown

---

#### contract_health
From Lens 2:

- weak → any P0/P1 contract finding
- medium → only P2 contract findings
- strong → explicit adapter + no contract findings
- unknown → no evidence

---

#### isolation_posture
From Lens 4:

- weak → no timeout/bulkhead/idempotency protection
- medium → partial isolation
- strong → explicit isolation controls
- unknown → no evidence

---

#### regionalization_blocker

Classification:

hard_blocker if:
- hard-critical AND (contract_health=weak OR isolation_posture=weak)
- OR any mapped P0 finding

soft_blocker if:
- soft-critical AND (weak contract OR weak isolation)

enabler if:
- findings strengthen boundaries without structural blockers

unknown otherwise.

All classifications must reference source findings.

---

## Outputs

Write under:

resilience/regionalization/

---

### 1) regionalization_inventory.csv (Primary Output)

Each row represents a single dependency.

Columns:

-----------------------------------------
Identity
-----------------------------------------
repo
role
dependency
dependency_type

-----------------------------------------
Operational Surface
-----------------------------------------
impacted_entrypoints
call_paths_observed
evidence_refs

-----------------------------------------
Regionalization Posture
-----------------------------------------
runtime_criticality
global_ok
must_be_local
contract_health
isolation_posture

-----------------------------------------
Gaps Identified
-----------------------------------------
contract_gaps
plane_separation_gaps

-----------------------------------------
Implementation Detail
-----------------------------------------
implementation_summary
implementation_steps
implementation_scope
risk_if_not_done
validation_strategy

-----------------------------------------
Regionalization Classification
-----------------------------------------
regionalization_blocker
priority
effort
confidence

-----------------------------------------
Traceability
-----------------------------------------
source_findings
open_questions

All unknown fields must be explicitly marked `unknown`.

---

## Implementation Detail Expansion Rules

Implementation fields must be derived from mapped findings.

No new recommendations may be invented.

### implementation_summary
1–2 sentence summary of what must change.

### implementation_steps
Concrete actionable steps, collapsed into semicolon-separated format:
- introduce adapter
- split control vs operational table
- add timeout config
- isolate executor
- introduce cache/LKG
- etc.

Must be implementable.

### implementation_scope
One of:
- code-only
- schema-change
- config-change
- cross-repo
- unknown

Derived from improvement description.

### risk_if_not_done
Concrete failure scenario derived from customer impact:
- cross-region outage cascades
- latency amplification
- blocking request path
- data residency violation
- etc.

Must reflect observable behavior.

### validation_strategy
Derived from lens validation:
- simulate dependency outage
- inject latency
- disable config store
- test idempotency under retry
- etc.

---

### 2) open_questions.md

List:

- Unmapped dependencies
- Missing lens coverage
- Unknown posture classifications
- Evidence gaps blocking decision
- Dependencies referenced in findings but not in discovery

Each question must:
- Reference dependency
- Reference evidence gap
- Explain why clarification matters

---

### 3) readme.md

Include:

- Repo name
- Role
- Number of dependencies analyzed
- hard_blockers count
- soft_blockers count
- enablers count
- unknown classifications count
- Lenses consumed
- Missing lens coverage

This file provides leadership-level snapshot.

---

## Evidence Rules (Inherited)

- No new findings.
- No topology inference.
- No region deployment design.
- No speculative classification.
- All derived posture must trace to source findings.

Unknown is acceptable.

---

## Phase Boundaries

This command:

- Produces synthesis artifacts only
- Does not propose multi-region design
- Does not introduce new deployment units
- Feeds Phase 2 regionalization planning

---

## Output Quality Bar

This command is successful when:

- Every dependency has posture classification or explicit unknown
- All blockers are traceable to evidence
- Implementation steps are actionable
- Leadership can answer:
    - “What blocks adding a region?”
    - “What must we fix first?”
    - “What happens if we do nothing?”