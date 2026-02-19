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

Expectations (best effort):
- If lens backlog files are present, they SHOULD conform to `skills/lenses.md` and include:
    - `FindingId`
    - `DependencyIds`
    - `EntrypointIds`
    - `Plane`

If a lens output is missing or malformed:
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
- entrypoints (including ids and bindings)
- dependencies (including dependency_id, call_sites, call_path)
- regionalization_facts (data_assets_observed)
- open questions

Build internal maps:
- dependency_id -> dependency record
- entrypoint_id -> entrypoint record
- dependency_id -> call_path(s) observed
- dependency_id -> discovery evidence refs (call_sites)

No inference beyond discovery artifacts.

---

### 2) Load Lens Findings

For each lens backlog file present:

Normalize each finding into:

- lens
- finding_id
- finding description
- severity
- confidence
- evidence
- improvement
- effort
- notes
- dependency_ids (from `DependencyIds`)
- entrypoint_ids (from `EntrypointIds`)
- plane (from `Plane` when explicitly evidenced; else `unknown`)

Preserve traceability to original lens outputs.

If backlog file is malformed:
- Mark lens coverage as `unknown`
- Add note to `open_questions.md`

---

### 3) Map Findings to Dependencies (Deterministic)

For each dependency in discovery, attach findings using the following precedence:

#### 3.1 Primary mapping (deterministic)
If a finding includes `DependencyIds` (not `UNKNOWN`):
- Map the finding to each referenced dependency_id exactly.

#### 3.2 Secondary mapping (fallback; evidence-backed only)
Only if `DependencyIds` is missing or `UNKNOWN`, map a finding to a dependency when the finding contains explicit evidence that matches discovery artifacts, such as:
- dependency display name exactly as listed in discovery
- explicitly cited datastore identifiers from `regionalization_facts.data_assets_observed[]`
- explicitly cited client/SDK/call-site file paths that appear in `dependencies[].call_sites[]`

Rules:
- No heuristic inference beyond explicit matches.
- If mapping is ambiguous, do not map; record an open question.

#### 3.3 Coverage guarantee note
If no findings map to a dependency:
- The dependency remains in scope and MUST still be emitted in the inventory with posture fields set to `unknown`.
- Add a per-dependency note in `open_questions.md` indicating “no mapped findings.”

---

### 4) Derive Regionalization Posture (Deterministic, Traceable)

All posture fields must be derived strictly from mapped findings (when present).
If no mapped findings exist for a dependency, posture fields MUST be `unknown`.

#### runtime_criticality
Source:
- From Lens 4 (Dependency Criticality and Isolation) classification findings only.
  Else `unknown`.

Values:
- hard
- soft
- non
- unknown

#### global_ok / must_be_local
Source:
- From Plane Separation lens findings only when they explicitly state:
    - a dependency must be local / is unsafe to call remotely on the hot path, OR
    - global is acceptable only with caching/LKG/degradation.

Values:
- global_ok: yes | conditional | no | unknown
- must_be_local: yes | no | unknown

Rules:
- If the lens does not explicitly make a locality claim, set to `unknown`.
- This command must not infer locality from dependency type alone.

#### contract_health
Source:
- From Domain Decoupling findings only.

Values:
- weak: any mapped P0/P1 contract finding
- medium: only mapped P2 contract findings
- strong: explicit adapter boundary and no contract findings
- unknown: no evidence

#### isolation_posture
Source:
- From Dependency Isolation findings only.

Values:
- weak: mapped findings indicate missing/insufficient timeouts/circuit breakers/bulkheads/idempotency
- medium: partial isolation improvements present
- strong: explicit isolation controls present and no isolation findings
- unknown: no evidence

#### regionalization_blocker
Classification (derived; must be traceable to source findings):

- hard_blocker if:
    - any mapped P0 finding exists, OR
    - runtime_criticality=hard AND (contract_health=weak OR isolation_posture=weak)
- soft_blocker if:
    - runtime_criticality=soft AND (contract_health=weak OR isolation_posture=weak)
- enabler if:
    - mapped findings exist and are primarily boundary-strengthening without blocker criteria
- unknown otherwise

All classifications MUST include references to the source finding IDs that triggered the classification.

---

## Outputs

Write under:

resilience/regionalization/

---

### 1) regionalization_inventory.csv (Primary Output)

Each row represents a single dependency from discovery (coverage guaranteed).

Columns:

-----------------------------------------
Identity
-----------------------------------------
repo
role
dependency_id
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
isolation_gaps

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

Rules:
- `impacted_entrypoints` MUST be derived from mapped `EntrypointIds` when present.
- If no `EntrypointIds` are mapped for a dependency, set `impacted_entrypoints=unknown` and cite discovery evidence refs.

---

### 2) open_questions.md

List:
- Unmapped dependencies (no findings mapped)
- Missing lens coverage
- Unknown posture classifications
- Evidence gaps blocking decision
- Dependencies referenced in findings but not in discovery
- Findings with `DependencyIds=UNKNOWN` or `EntrypointIds=UNKNOWN`

Each question must:
- Reference dependency_id (when applicable)
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
- add circuit breaker
- cap retries / add time budget
- add idempotency key reuse
- etc.

Must be implementable and derived from the findings’ Improvement text.

### implementation_scope
One of:
- code-only
- schema-change
- config-change
- cross-repo
- unknown

Derived from improvement description; if ambiguous, set `unknown`.

### risk_if_not_done
Concrete failure scenario derived from customer impact:
- cross-region outage cascades
- latency amplification
- blocking request path
- duplicate side effects under retry
- stale status updates
- etc.

Must reflect observable behavior and be traceable to findings.

### validation_strategy
Derived from lens validation:
- simulate dependency outage
- inject latency
- disable config store
- test idempotency under retry
- verify circuit breaker opens/closes
- etc.

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
- Every dependency from discovery has a posture classification or explicit `unknown`
- All blockers are traceable to evidence (finding IDs + evidence refs)
- Implementation steps are actionable and derived from findings
- Leadership can answer:
    - “What blocks adding a region?”
    - “What must we fix first?”
    - “What happens if we do nothing?”