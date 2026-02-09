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
•	Source of truth for current architecture
•	Reflects discovery facts only
•	No inferred behavior or proposed changes
•	Generated from resilience/discovery/

### arch-after.mmd (Per Lens)

arch-after.mmd is used to visualize Phase 1 internal boundary changes within the existing deployment topology.
It is not a future-state or topology diagram.

#### Lens 1: Plane Separation (Control vs Operational)

Generate arch-after.mmd when findings include:
•	separating control-plane reads/writes from operational execution paths
•	introducing async control refresh instead of synchronous calls
•	isolating control-plane persistence or caches from operational traffic
•	separate executors or request paths for control vs operational work

Diagram should show:
•	explicit control-plane vs operational-plane boundaries
•	which components move off the request path
•	persistence or cache separation (conceptual)

Do NOT show:
•	new services
•	regional placement
•	control-plane ownership changes

⸻

#### Lens 2: Domain Decoupling via Explicit Interfaces

Generate arch-after.mmd when findings include:
•	introducing adapters, facades, or anti-corruption layers
•	consolidating multiple integration call sites into a single boundary
•	normalizing external contracts or error semantics
•	removing direct dependency on upstream internals

Diagram should show:
•	explicit adapter/facade boundaries
•	Verify-owned interfaces vs external domains
•	directionality of dependencies

This lens is the strongest candidate for arch-after diagrams.

⸻

#### Lens 3: Monolith → Modulith

Generate arch-after.mmd when findings include:
•	identifying durable internal subsystems (routing, verification core, fraud, provider integration)
•	isolating execution resources (executors, queues, caches) per subsystem
•	removing hidden propagation paths between concerns

Diagram should show:
•	internal module boundaries within the role
•	ownership and dependency direction
•	failure isolation points

Do NOT generate if findings are purely code hygiene or refactoring suggestions without boundary implications.

⸻

#### Lens 4: Dependency Criticality & Isolation

Generate arch-after.mmd ONLY when findings include:
•	introducing bulkheads or per-dependency executors
•	moving non-critical dependencies off the request path
•	restructuring dependency access patterns (sync → async, cached → remote)

Diagram should show:
•	dependency isolation points
•	critical vs non-critical paths
•	where degradation or fail-open occurs

Do NOT generate for:
•	timeout tuning
•	retry policy changes
•	client configuration-only improvements

(This lens is usually behavioral, not structural.)

⸻

#### Global Rules (Apply to All Lenses)
•	No new deployment units
•	No regions or cells
•	No failover or HA semantics
•	No inferred topology changes
•	Boundaries must be inside the existing role

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