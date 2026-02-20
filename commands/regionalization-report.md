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
    - `EndpointBindings`
    - `Plane`
    - `BoundaryType`
    - `BoundaryName`
    - `AdapterNames`
    - `CustomerImpact`
    - `Improvement`
    - `Notes` (optional)
    - `Evidence` (optional but preferred)
    - `Confidence` (optional but preferred)
    - `Severity` (optional but preferred)

If a lens output is missing or malformed:
- Proceed with available inputs
- Mark coverage gaps as `unknown`
- Emit a validation note in `open_questions.md`

---

## Scope

This command operates **repo-locally only**.

It produces:
- A dependency-first regionalization inventory (Table A, CSV)
- A findings-first implementation detail inventory (Table B, CSV)
- Traceability back to original findings
- Explicit gaps and open questions

It does not aggregate across repositories.

---

## CSV Formatting Rules (Non-Negotiable)

All CSV outputs MUST be RFC4180-compatible:
- Use comma `,` as delimiter.
- Always output a header row.
- Any field containing comma, quote, or newline MUST be quoted with double quotes.
- Double quotes inside fields MUST be escaped as `""`.
- Newlines inside fields MUST be normalized to a single space (preferred) or `\n` (must be consistent).

---

## Execution Steps (In Order)

### 1) Load Discovery

Parse:
- role metadata
- operational_entrypoints (including ids and bindings/paths)
- outbound_dependencies (including dependency_id, call_sites, call_path)
- regionalization_facts (data_assets_observed)
- open questions

Build internal maps:
- dependency_id -> dependency record
- entrypoint_id -> entrypoint record
- dependency_id -> call_path(s) observed (from dependency call sites)
- dependency_id -> discovery evidence refs (call_sites)

No inference beyond discovery artifacts.

---

### 2) Load Lens Findings

For each lens backlog file present:

Normalize each finding into a canonical finding record with fields (best effort):
- FindingId
- Lens
- Finding (description)
- Severity
- Confidence
- Evidence
- CustomerImpact
- Improvement
- Notes
- DependencyIds (comma-separated list or UNKNOWN)
- EntrypointIds (comma-separated list or UNKNOWN)
- EndpointBindings (comma-separated list or UNKNOWN)
- Plane (control/operational/mixed/unknown)
- BoundaryType
- BoundaryName
- AdapterNames

Preserve traceability to original lens outputs.

If backlog file is malformed or missing:
- Mark lens coverage as `unknown`
- Add note to `open_questions.md`

---

### 3) Map Findings to Dependencies (Deterministic)

Attach findings to dependencies using:

#### 3.1 Primary mapping (deterministic)
If a finding includes `DependencyIds` (not `UNKNOWN`):
- Map the finding to each referenced dependency_id exactly.

#### 3.2 Secondary mapping (fallback; evidence-backed only)
Only if `DependencyIds` is missing or `UNKNOWN`, map a finding to a dependency when the finding contains explicit evidence that matches discovery artifacts, such as:
- dependency display name exactly as listed in discovery
- explicitly cited datastore identifiers from `regionalization_facts.data_assets_observed[]`
- explicitly cited client/SDK/call-site file paths that appear in `outbound_dependencies[].call_sites[]`

Rules:
- No heuristic inference beyond explicit matches.
- If mapping is ambiguous, do not map; record an open question.

#### 3.3 Coverage guarantee note
If no findings map to a dependency:
- The dependency remains in scope and MUST still be emitted in Table A with `Findings (IDs)=—` and `Max Sev=—`.
- Add a per-dependency note in `open_questions.md` indicating “no mapped findings.”

---

### 4) Derive Implementation Detail Columns (For Table B Only)

For each finding, derive the following fields strictly from the finding content.
No new recommendations may be invented.

#### implementation_summary
- 1 sentence derived from Improvement/Recommended Changes.

#### implementation_steps
- Semicolon-separated steps derived from Improvement/Recommended Changes.
- Must be implementable and traceable to finding text.

#### implementation_scope
One of:
- code-only
- config-change
- schema-change
- cross-repo
- unknown

Derivation rules (deterministic):
- code-only: refactors, interfaces/adapters, caches, async execution, bulkheads implemented in this repo
- config-change: timeout/CB/retry config changes with no code changes required (or explicitly config-focused findings)
- schema-change: new table/topic/queue or schema evolution required (e.g., DLQ, new stream entity)
- cross-repo: requires implementing a new consumer or changes in another service/repo
- unknown: insufficient evidence

#### risk_if_not_done
- 1 sentence derived strictly from CustomerImpact / “Why customers care”.

---

## Outputs

Write under:

resilience/regionalization/

---

### 1) dependency_rollup.csv (Table A)

A CSV file with **one row per dependency** from discovery (coverage guaranteed).

Header columns (exact):
Repo,DependencyId,DependencyName,Type,Call path (discovery),Endpoints impacted (full),Findings (IDs),Max Sev

Rules:
- `Repo` = discovery role name
- `Call path (discovery)` = deduped call_path values from dependency call sites (stable ordering)
- `Endpoints impacted (full)`:
    - Prefer mapped findings’ EndpointBindings if present
    - Else map EntrypointIds -> discovery bindings/paths
    - Else `UNKNOWN` (do not infer)
- `Findings (IDs)` = comma-separated finding IDs or `—`
- `Max Sev` = computed max severity (P0>P1>P2) or `—`

---

### 2) findings_detail.csv (Table B)

A CSV file with **one row per finding** across all lenses.

Header columns (exact):
FindingId,Lens,DependencyIds,Endpoints impacted (full),Coupling type / failure mechanics,Customer impact,Severity,Confidence,Evidence (file:line/range),Current mitigations,Recommendation (explicit interface / decoupling / isolation),implementation_summary,implementation_steps,implementation_scope,risk_if_not_done,Validation / Acceptance criteria,BoundaryType,BoundaryName,AdapterNames / Implementations

Rules:
- `Coupling type / failure mechanics` MUST be a concise extraction from finding text (no new claims).
- `Current mitigations` MUST be what the finding states; else `—`.
- `Validation / Acceptance criteria` must be derived from validation steps in the finding; else `—`.
- Boundary fields MUST be copied verbatim when present; else `—`.
- Implementation detail fields MUST be derived from finding text; no invention.

---

### 3) open_questions.md

List:
- Unmapped dependencies (no findings mapped)
- Missing lens coverage
- Findings with `DependencyIds=UNKNOWN` or `EntrypointIds=UNKNOWN` or `EndpointBindings=UNKNOWN`
- Evidence gaps blocking deterministic mapping (e.g., unknown queue/topic names)

Each question must:
- Reference dependency_id (when applicable)
- Reference evidence gap
- Explain why clarification matters

---

### 4) readme.md

Include:
- Repo name
- Role
- Number of dependencies analyzed
- Number of findings loaded
- Lenses consumed
- Missing lens coverage
- Count of dependencies with no mapped findings
- Count of findings with UNKNOWN dependency mapping

This file provides leadership-level snapshot.

---

## Evidence Rules (Inherited)

- No new findings.
- No topology inference.
- No region deployment design.
- No speculative classification.

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
- Every dependency from discovery appears in dependency_rollup.csv
- Every finding from available lens backlogs appears in findings_detail.csv
- All non-UNKNOWN fields are traceable to source artifacts
- Implementation fields in Table B are derived from finding text
- Leadership can answer:
    - “What blocks adding a region?”
    - “What must we fix first?”
    - “What happens if we do nothing?”