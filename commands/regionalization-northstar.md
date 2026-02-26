# Command: /regionalization-northstar

## Intent

Generate a **per-repo**, evidence-backed **regionalization north star narrative** that connects:

1) **Data residency readiness** — identify repo-local data assets that act as *residency anchors*
   and the couplings that prevent region-scoped data ownership.

2) **Resiliency readiness** — identify repo-local failure-domain couplings that make operating
   multiple regions harder (blast radius, control-plane coupling, shared resources).

This command is **additive** to existing Phase 1 artifacts. It is a **synthesis and planning layer**
that consumes discovery + lens outputs. It does **not** re-analyze source code.

This command answers:

> “For this repo, what must be true (internally) to support adding regions later,
> and what is the phased plan to get there, grounded in existing findings and discovery?”

---

## Inputs

Required:
- `resilience/discovery/discovery.json`

Optional (best effort if present):
- `resilience/role_dependency_table.csv`
- `resilience/regionalization/findings_detail.csv` (from `/regionalization-report`)
- `resilience/lens_plane-separation/backlog.csv`
- `resilience/lens_domain-decoupling/backlog.csv`
- `resilience/lens_modulith/backlog.csv`
- `resilience/lens_dependency-isolation/backlog.csv`

Expectations (best effort):
- If lens backlog files are present, they SHOULD conform to `skills/lenses.md` and include:
  `FindingId, Lens, Severity, Confidence, Evidence, WhyCustomersCare (or CustomerImpact),
   RecommendedChange (or Improvement), ValidationSteps, EntrypointIds, EndpointBindings,
   DependencyIds, Plane, BoundaryType, BoundaryName, AdapterNames`

If required inputs are missing:
- If `discovery.json` is missing, abort with an explicit error explaining that this command
  is synthesis-only and requires discovery output.
- If optional inputs are missing, proceed and mark coverage gaps in the output (UNKNOWN is acceptable).

---

## Scope

Repo-local only.

This command:
- **MAY** propose *candidate internal boundaries* (modulith seams, adapters, cache boundaries, bulkheads)
  **only** when directly supported by existing findings’ `BoundaryType/BoundaryName` and/or
  evidenced coupling described in findings.
- **MAY** sequence work into phases/epics and define acceptance criteria by reusing finding validation steps.
- **MAY** generate a repo-local, cross-lens boundary diagram (`arch-northstar.mmd`) that visualizes
  the target internal seams and dependency direction (no topology).
- **MUST NOT**:
    - infer or propose multi-region topology (active-active, failover, traffic routing, cells)
    - assert where data resides in production regions
    - create new findings, severities, or recommendations not present in inputs
    - reclassify dependencies beyond what discovery provides

Unknown is acceptable.

---

## Evidence Rules (Non-Negotiable)

- No new findings.
- No facts without evidence.
- Every north star statement MUST be traceable to at least one of:
    - a finding row (FindingId + Evidence), and/or
    - a discovery.json fact with evidence (file:line/range).
- If traceability is not possible, mark as `UNKNOWN` in outputs.

---

## Execution Steps (In Order)

### 1) Load Discovery (Authoritative Facts)

Parse `resilience/discovery/discovery.json` and build internal maps:

- role metadata: `role`, `owner_team`
- `operational_entrypoints[]`
    - id → binding/route/evidence
    - also derive a normalized endpoint binding string when possible:
        - if discovery has `binding` + `route`, normalize to `"<METHOD> <route>"`
        - else use `route` or `binding` as available and mark unknowns
- `control_plane_ops[]`
    - id → type, invocation_context, evidence
- `outbound_dependencies[]`
    - dependency_id → {name, type, call_path_classification, call_sites, data_assets[]}
- `regionalization_facts.data_assets_observed[]`
    - asset_id → {engine, type, identifier, used_on_paths, evidence}

Do not infer anything beyond discovery facts.

### 2) Load Findings (Lens Backlogs and/or findings_detail)

Load findings from the best available sources, in this priority order:

1) `resilience/regionalization/findings_detail.csv` (if present)
2) lens backlog CSVs (if present)
3) otherwise: no findings loaded (still emit outputs but with gaps)

Normalize into a canonical finding record:

- FindingId
- Lens
- Severity, Confidence
- Evidence (verbatim)
- CustomerImpact (WhyCustomersCare/CustomerImpact)
- Recommendation (RecommendedChange/Improvement)
- ValidationSteps (verbatim)
- EntrypointIds, EndpointBindings, DependencyIds
- Plane
- BoundaryType, BoundaryName, AdapterNames

Rules:
- Preserve traceability to original artifacts (include file paths and line ranges verbatim).
- If fields are absent, set `UNKNOWN` or `—` as appropriate.

### 3) Deterministic Mapping: Findings → Dependencies → Data Assets

Attach findings to dependencies:

#### 3.1 Primary mapping (deterministic)
If `DependencyIds` is present and not `UNKNOWN`:
- Map the finding to each referenced dependency_id exactly.

#### 3.2 Secondary mapping (fallback; evidence-backed only)
Only if `DependencyIds` is missing or `UNKNOWN`, map a finding to a dependency when the finding
contains explicit text or evidence that matches discovery artifacts, such as:
- dependency_id or dependency display name as listed in discovery
- explicitly cited datastore/table/topic/queue identifiers that match `regionalization_facts.data_assets_observed[].identifier`
- explicitly cited file paths that match discovery `outbound_dependencies[].call_sites[].file`

No heuristic inference beyond explicit matches.
If ambiguous, do not map.

Attach dependencies to data assets using discovery only:
- dependency.data_assets[] → asset records (if present)
- if missing, do not infer linkage (keep UNKNOWN where needed).

### 4) Derive Repo-Local Inventories (No New Findings)

#### 4.1 Residency anchors (data assets)
A data asset is a “residency anchor candidate” if discovery shows:
- asset_type in `{table, stream, queue, cache, bucket, unknown}` AND
- engine in `{dynamodb, redis, kafka, sqs, s3, unknown}` AND
- identifier is present (not `unknown`) OR evidence exists of its usage as a concrete asset even if identifier is unknown.

For each asset, compute:
- which dependencies reference it (from discovery dependency.data_assets)
- which entrypoint paths touch those dependencies (from mapped findings; else UNKNOWN)
- max severity of mapped findings affecting the asset’s dependency paths

No additional classification (e.g., PII) is allowed.

#### 4.2 Service dependency anchors (internal/external services)
A service dependency anchor is any outbound dependency where:
- dependency_type in `{internal_service, third_party_api}`

For each service dependency, compute:
- call_path_classification (from discovery)
- endpoints impacted (from mapped findings; else UNKNOWN)
- max severity + key finding IDs

Do not infer “criticality”; only roll up what findings say.

### 5) Build a North Star Roadmap (Bounded Synthesis)

Create 4–7 **epics** that represent repo-local prerequisites for regionalization.

Rules:
- Each epic MUST be backed by one or more findings.
- Each epic MUST state:
    - goal (north-star outcome)
    - what boundary/seam it introduces (BoundaryType/BoundaryName, if present)
    - which residency anchors / dependencies it affects (from discovery)
    - why it matters for residency and/or resiliency (derived from finding impact text)
    - acceptance criteria (derived from finding ValidationSteps; do not invent)

Phasing:
- Phase 0: “Stop-the-bleeding” reliability/operability prerequisites (timeouts, non-blocking, per-channel CB)
- Phase 1: Introduce durable seams (module boundaries, adapters, repository interfaces/projections)
- Phase 2: Hardening for scale (bulkheads, bounded queues, async workers) when already present in findings

Do not introduce topology changes.

### 6) Generate `arch-northstar.mmd` (Cross-Lens Boundary Diagram)

Generate `arch-northstar.mmd` if (and only if) at least one loaded finding has `BoundaryType` in:
`gateway, facade, adapter, port, anti_corruption_layer, module_seam, cache_boundary, bulkhead, outbox, published_events, read_path_optimization, internal_events, module_interface`

Purpose:
- Visualize the **candidate internal seams and dependency direction** implied by findings across lenses.
- Provide a single “end-state within one deployable” view (NOT a topology diagram).

Non-negotiable constraints:
- No new deployment units.
- No regions/cells/failover/active-active semantics.
- No inferred topology.
- Only include boundary nodes that are backed by findings.

#### 6.1 Node admissibility rule (no invention)
A boundary/module node may appear only if backed by at least one finding via:
- explicit `BoundaryType` + `BoundaryName`, and/or
- explicit extraction recommendation naming the boundary (e.g., “Extract RouteSelectionService”)
  AND that finding has evidence.

Each boundary node label MUST include `BackedBy: <FindingIds>`.

#### 6.2 Edge admissibility rule (no invented coupling)
Draw an edge only when supported by:
- discovery outbound dependency call sites / call_path_classification, and/or
- finding text describing the coupling (with evidence).

#### 6.3 Mermaid syntax rules
All Mermaid output MUST comply with the Mermaid rules in `skills/reporting.md`:
- sanitized node IDs (`N001`, `N002`, …)
- human-readable detail in labels only
- no file paths as node IDs
- normalize quotes/newlines

---

## Outputs

Write under:
`resilience/regionalization/`

This command produces **exactly three files**:
1) `northstar.md` (primary narrative)
2) `northstar.csv` (single structured table for cross-repo/portfolio use)
3) `arch-northstar.mmd` (conditional; generate only when warranted by boundary-type findings; otherwise omit)

---

### 1) `northstar.md` (primary narrative)

Required sections (use these exact headings):

#### A. Repo Overview (Facts)
- Role name
- Owner team (if present)
- Key operational entrypoints (id + normalized binding)
- Key control-plane ops (id + invocation_context)
- Outbound dependency count and data asset count

All factual; sourced from discovery.

#### B. What “Regionalization-Ready” Means for This Repo (Invariants)
List 6–10 invariants split into:
- Data residency invariants (repo-local)
- Resiliency invariants (repo-local)
- Modulith boundary invariants (repo-local)

For each invariant include:
- Statement (1 sentence)
- BackedBy (FindingIds and/or discovery evidence refs)

#### C. Residency Anchors (Evidence-backed Inventory)
Table (markdown) with columns:
- AssetId
- Engine
- Type
- Identifier
- UsedOnPaths (from discovery)
- Dependent Dependencies (dependency_ids)
- Findings (IDs)
- Max Severity
- Evidence

No inference beyond discovery + mapped findings.

#### D. Service Dependency Anchors (Internal + External)
Table (markdown) with columns:
- DependencyId
- DependencyName
- Type (`internal_service` or `third_party_api`)
- CallPathClassification (from discovery)
- Endpoints impacted (full) (from mapped findings; else `UNKNOWN`)
- Findings (IDs)
- Max Severity
- Evidence (discovery call sites + top finding evidence)

No inference; roll up only.

#### E. Regionalization Blockers and Enablers (By Epic)
For each epic:
- Goal
- Why it matters (Residency + Resiliency)
- Backed by (FindingIds)
- Key evidence (top 2–5 evidence refs)
- Proposed boundary/seam (BoundaryType/BoundaryName, verbatim when present)
- Milestones (phase steps; derived from recommended changes)
- Acceptance criteria (derived from validation steps)

#### F. Phased Roadmap (Repo-local)
Provide a short phase plan:
- Phase 0 / Phase 1 / Phase 2
  Each phase includes:
- Epics included
- Preconditions
- Primary risks if deferred (derived from customer impact)

#### G. Open Questions (Blocking Regionalization Decisions)
Include:
- Discovery open_questions (verbatim)
- Any mapping gaps encountered in this command
- Any residency anchor with unknown identifier or unknown usage linkage

Each question must state:
- what is unknown
- which decision it blocks (residency, resiliency, sequencing)
- where to look (file/config area)

---

### 2) `northstar.csv` (single structured rollup)

RFC4180-compatible CSV. One row per **North Star Epic**.

Header columns (exact):
Repo,EpicId,EpicName,Phase,Goal,Why (residency),Why (resiliency),PrimaryAssets,PrimaryDependencies,PrimaryEntrypoints,BackedByFindingIds,BoundaryTargets,KeyEvidence,Milestones,AcceptanceCriteria,OpenQuestions

Rules:
- `Repo` = discovery role name
- `Phase` = `0` | `1` | `2` | `unknown`
- `PrimaryAssets` = comma-separated asset_ids (from discovery) or `—`
- `PrimaryDependencies` = comma-separated dependency_ids (from discovery) or `—`
- `PrimaryEntrypoints` = comma-separated entrypoint ids (from discovery) or `—`
- `BackedByFindingIds` = comma-separated finding IDs or `—`
- `BoundaryTargets` = concatenated `BoundaryType:BoundaryName` (include AdapterNames when present), or `—`
- `KeyEvidence` = semicolon-separated evidence refs (file:line/range) from findings and/or discovery
- `Milestones` and `AcceptanceCriteria` must be derived from findings’ recommended changes and validation steps (no invention)
- `OpenQuestions` = semicolon-separated discovery question ids and/or concise text

---

### 3) `arch-northstar.mmd` (Conditional)

Generate under the conditions in Execution Step 6.

Diagram content rules:
- Must represent **internal boundaries** and **dependency direction** only.
- Must show:
    - role boundary
    - grouped operational entrypoints (conceptually) by candidate module where supported by findings
    - control-plane ops separated conceptually from operational path when evidenced by findings
    - external dependencies and data assets (from discovery) connected to the relevant boundary nodes when supported
- Must not show:
    - regions/cells/topology
    - failover semantics
    - SLO/SLA claims

Traceability rule:
- Every boundary node label includes `BackedBy: FindingIds`.
- Every external dependency node label includes discovery evidence reference(s).

---

## CSV Formatting Rules (Non-Negotiable)

All CSV outputs MUST be RFC4180-compatible:
- Use comma `,` as delimiter.
- Always output a header row.
- Any field containing comma, quote, or newline MUST be quoted with double quotes.
- Double quotes inside fields MUST be escaped as `""`.
- Newlines inside fields MUST be normalized to a single space (preferred) or `\n` (must be consistent).

---

## Output Quality Bar

This command is successful when:
- `northstar.md` is understandable on its own and contains the residency + resiliency story.
- `northstar.csv` enables cross-repo/portfolio aggregation without re-reading narrative docs.
- Every epic references at least one FindingId and has evidence citations.
- No epic introduces recommendations not present in findings.
- Unknowns are explicitly listed and tied to decision impact.
- If `arch-northstar.mmd` is generated, it contains no invented boundaries and complies with Mermaid rules.
- Leadership can answer, per repo:
    - “What data assets are the residency anchors here?”
    - “What internal seams must exist before adding regions is feasible?”
    - “What is the phased plan and how do we validate progress?”

---

## Questions (Must Ask When Missing)

If any of the following are true, ask the user before finalizing outputs:
1) No lens findings are available (only discovery exists).
2) Findings exist but most have `DependencyIds=UNKNOWN`.
3) The repo has multiple subprojects and it’s unclear which `resilience/` directory is authoritative.

When asking, request only the minimal missing artifact(s), such as:
- the specific missing backlog.csv file(s), or
- the `role_dependency_table.csv`, or
- the `resilience/regionalization/findings_detail.csv`.