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

Each row MUST include the fields defined in `skills/lenses.md`, including:
- EntrypointIds
- EndpointBindings (full bindings, not IDs only)
- DependencyIds
- Plane
- BoundaryType, BoundaryName, AdapterNames

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

### 5) arch-after.mmd (Conditional; strongly preferred when warranted)

`arch-after.mmd` is used to visualize Phase 1 internal boundary changes within the existing deployment topology.
It is not a future-state or topology diagram.

Written to:
- `resilience/lens_<lens_name>/arch-after.mmd`

#### Generate when (default-on)
Generate `arch-after.mmd` when the lens produces at least one finding that introduces a **boundary/seam** or **execution isolation point**, such as:
- introducing adapters, facades, registries, ports/anti-corruption layers (Domain Decoupling)
- moving control-plane reads off the request path via caching/LKG/background refresh (Plane Separation)
- splitting shared executors/thread pools into per-flow pools (Plane Separation or Modulith)
- extracting internal modules/subsystems with clear ownership and dependency direction (Modulith)
- adding bulkheads / per-dependency executors / bounded queues / outbox-like decoupling (Dependency Isolation)

#### Do NOT generate when
Do NOT generate `arch-after.mmd` when the lens produces only behavioral/config changes that do not introduce a boundary or isolation point, such as:
- timeout tuning only
- retry policy tuning only
- circuit breaker threshold tuning only
- logging/metrics-only improvements

#### Output completeness rule (non-negotiable)
If a lens backlog.csv contains any row with `BoundaryType` in:
`gateway, facade, adapter, port, anti_corruption_layer, module_seam, cache_boundary, bulkhead, outbox`
then `arch-after.mmd` MUST be generated for that lens.
If not generated, the lens output is considered incomplete.

#### Diagram content rules
- Show only **internal boundaries** and **dependency direction** within the same role.
- Do not show new deployment units, regions, cells, failover semantics, or HA guarantees.
- Diagrams must remain conceptual but must be traceable to findings.

---

## Additional Output (Per Role): Dependency-First Rollup Table (Mechanical Merge)

### Purpose
Produce a single dependency-first table per role/deployment unit to support planning and
regionalization discussions. This artifact is a deterministic merge of discovery inventory
and lens findings. It introduces NO new findings and performs NO new analysis.

### Output Location
`resilience/role_dependency_table.csv`

### Inputs
- `resilience/discovery/discovery.json`
- `resilience/lens_plane-separation/backlog.csv`
- `resilience/lens_domain-decoupling/backlog.csv`
- `resilience/lens_modulith/backlog.csv`
- `resilience/lens_dependency-isolation/backlog.csv`

### Non-Negotiable Rules
- The table MUST NOT introduce new findings, severities, or recommendations.
- Each row MUST be traceable to either:
    1) a discovered dependency (inventory row), or
    2) a specific lens finding row joined to dependency and entrypoint IDs/bindings.
- If a dependency has no lens findings, it MUST still appear as an Inventory row.

### Row Guarantee (Coverage Requirement)
For every entry in `discovery.json.dependencies[]`, emit at least one row.

If no findings map to the dependency:
- `Lenses = Inventory`
- `FindingId = —`
- `Severity = —`
- `Confidence = —`
- `Improvement = —`
- `Evidence = (dependency call_sites evidence)`

### Required Columns
Each row MUST include:

| Column | Description |
|---|---|
| Role | Role / deployment unit |
| DependencyId | Canonical dependency id from discovery |
| DependencyName | Display name from discovery |
| DependencyType | internal_service / third_party_api / datastore / queue_or_stream / cache / stream / queue |
| CallPath | From discovery (`sync_operational` / `async` / `control_plane` / `unknown`) |
| PlaneRow | Derived from CallPath (`operational` for sync_operational/async, `control` for control_plane, else `unknown`) unless overridden by lens `Plane` when explicitly evidenced |
| EndpointBindings | Comma-separated full endpoint bindings (from backlog.csv `EndpointBindings`, or via EntrypointIds join), or `UNKNOWN` |
| Lenses | Comma-separated list of lenses with findings for this dependency, else `Inventory` |
| FindingIds | Comma-separated finding IDs, else `—` |
| MaxSeverity | Maximum severity across joined findings (computed only; no reclassification) |
| Improvements | Concatenated improvements from findings (verbatim), else `—` |
| BoundaryTargets | Concatenated `BoundaryType:BoundaryName` (and AdapterNames when present), else `—` |
| Evidence | Evidence from findings (verbatim) plus discovery call site evidence |
| Notes | Any validation questions from findings (verbatim) |

### Optional Columns (Regionalization Facts Only)
These columns may be included ONLY when directly evidenced in discovery/config:

- `DataAssetType`
- `DataAssetEngine`
- `DataAssetIdentifier`
- `ConfiguredRegion` (e.g., if SQS region is explicitly present)
- `ReplicationConfigEvidence` (e.g., terraform references)

Rules:
- These fields must be evidence-backed and must not infer topology.

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

### Mermaid Generation Rules (Non-Negotiable)

To avoid Mermaid syntax errors, all generated `.mmd` files must follow these rules:

**Node IDs**
- Node IDs MUST be sanitized and machine-safe.
- Allowed characters: `[A–Z][a–z][0–9]_`
- IDs MUST NOT contain spaces or special characters (`{ } ( ) [ ] : " ' < > /`)
- IDs MUST NOT encode human-readable information.
- Prefer simple deterministic IDs (e.g., `N001`, `N002`, …).

**Node Labels**
- Human-readable information (file paths, method names, service names) MUST appear only in labels.
- Labels MUST be quoted using square brackets: N001[“VerificationResource.java:200–204”]
- Quotes inside labels MUST be normalized (`"` → `'`).
- Newlines MUST be replaced with spaces.

**Edges**
- Avoid edge labels unless necessary.
- If used, edge labels MUST be short and free of special characters.

**Prohibited**
- Using file paths, method signatures, or URLs as node IDs.
- Using `{}` or `()` node shapes when labels contain punctuation.

All diagrams that violate these rules are considered invalid outputs.

### arch-before.mmd
• Source of truth for current architecture  
• Reflects discovery facts only  
• No inferred behavior or proposed changes  
• Generated from resilience/discovery/

### arch-after.mmd (Per Lens)
arch-after.mmd is used to visualize Phase 1 internal boundary changes within the existing deployment topology.
It is not a future-state or topology diagram.

#### Lens 1: Plane Separation (Control vs Operational)
Generate arch-after.mmd when findings include:
• separating control-plane reads/writes from operational execution paths  
• introducing async control refresh instead of synchronous calls  
• isolating control-plane persistence or caches from operational traffic  
• separate executors or request paths for control vs operational work

Diagram should show:
• explicit control-plane vs operational-plane boundaries  
• which components move off the request path  
• persistence or cache separation (conceptual)

Do NOT show:
• new services  
• regional placement  
• control-plane ownership changes

⸻

#### Lens 2: Domain Decoupling via Explicit Interfaces
Generate arch-after.mmd when findings include:
• introducing adapters, facades, or anti-corruption layers  
• consolidating multiple integration call sites into a single boundary  
• normalizing external contracts or error semantics  
• removing direct dependency on upstream internals

Diagram should show:
• explicit adapter/facade boundaries  
• Verify-owned interfaces vs external domains  
• directionality of dependencies

This lens is the strongest candidate for arch-after diagrams.

⸻

#### Lens 3: Monolith → Modulith
Generate arch-after.mmd when findings include:
• identifying durable internal subsystems (routing, verification core, fraud, provider integration)  
• isolating execution resources (executors, queues, caches) per subsystem  
• removing hidden propagation paths between concerns

Diagram should show:
• internal module boundaries within the role  
• ownership and dependency direction  
• failure isolation points

Do NOT generate if findings are purely code hygiene or refactoring suggestions without boundary implications.

⸻

#### Lens 4: Dependency Criticality & Isolation
Generate arch-after.mmd ONLY when findings include:
• introducing bulkheads or per-dependency executors  
• moving non-critical dependencies off the request path  
• restructuring dependency access patterns (sync → async, cached → remote)

Diagram should show:
• dependency isolation points  
• critical vs non-critical paths  
• where degradation or fail-open occurs

Do NOT generate for:
• timeout tuning  
• retry policy changes  
• client configuration-only improvements

(This lens is usually behavioral, not structural.)

⸻

#### Global Rules (Apply to All Lenses)
• No new deployment units  
• No regions or cells  
• No failover or HA semantics  
• No inferred topology changes  
• Boundaries must be inside the existing role

If a lens produces only behavioral changes, arch-after.mmd must not be generated.

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