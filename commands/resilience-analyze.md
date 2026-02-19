# Command: /resilience-analyze

## Intent
Analyze a role/service repository using the Resiliency Analysis Lenses and produce Phase 1 artifacts that are directly consumable for prioritization and execution.

This command must be evidence-backed, Phase 1 scoped, and must not assume topology changes.

---

## Inputs
- One or more repositories (local workspace).
- Optional: an existing discovery artifact produced by `/resilience-map`:
    - `resilience/discovery/discovery.json`

If discovery is not present:
- Run the equivalent discovery steps inline (best effort), limited to what is required by the selected lenses.
- Mark unknowns explicitly and emit validation questions.

---

## Parameters
- `lens`: one of `{plane-separation, domain-decoupling, modulith, dependency-isolation, all}` (default: `all`)
- `phase`: `1` (default and only supported)
- `flows`: N critical entrypoints to focus on (default: `5`)

---

## Execution Steps (Do These In Order)

### 1) Load or Generate Discovery
If `resilience/discovery/discovery.json` exists, load it.

Otherwise generate a minimal inline discovery model sufficient for the selected lenses:
- operational entrypoints (required)
- control-plane operations (best effort)
- dependencies and call sites (best effort)
- regionalization_facts (optional best effort)

Notes:
- Do not force virtual module inference during inline discovery.
- Unknowns must be captured as validation questions.

---

### 2) Select Critical Entrypoints / Flows
Select up to N critical operational entrypoints to anchor analysis:
- Prefer repo metadata “critical flows” if present.
- Otherwise select using best effort heuristics:
    - centrality (high fan-out)
    - business importance (send/check verification, fallback, callbacks/consumers)
    - frequency indicators (if detectable)

Record:
- selected entrypoints (with evidence)
- selection rationale

---

### 3) Apply Selected Lenses
Apply the requested lens set:
- Lens 1: Plane Separation (Control vs Operational)
- Lens 2: Domain Decoupling via Explicit Interfaces
- Lens 3: Monolith → Modulith (Lightweight, Improvement-Driven)
- Lens 4: Dependency Criticality and Isolation

For each lens finding, conform to:
- `skills/lenses.md` (lens-specific patterns + candidates)
- `skills/verification.md` (rigor rules)
- `skills/reporting.md` (output shape)

Each finding MUST include:
- severity: `P0` | `P1` | `P2`
- confidence: `High` | `Medium` | `Low`
- evidence: file + line range (or config key + location)
- impacted entrypoints (if applicable)
- why customers care (1 sentence)
- recommended change (1–3 bullets) OR investigation task (if evidence is insufficient)
- validation steps (1–3 bullets)

Structured join keys + boundary naming (required in `backlog.csv`):
- `FindingId` (stable within lens)
- `EntrypointIds` (from discovery entrypoints)
- `EndpointBindings` (full bindings copied verbatim from discovery; e.g., `HTTP POST /v1/.../Messages`)
- `DependencyIds` (from discovery outbound dependencies)
- `Plane` (control/operational/mixed/unknown when explicitly evidenced; else unknown)
- `BoundaryType`, `BoundaryName`, `AdapterNames` (per `skills/lenses.md`)

If evidence cannot be found:
- mark the finding as `UNKNOWN`
- emit a validation question instead of a recommendation

---

### 4) Convert Findings into Executable Backlog Items
For the selected lens(es):
- translate findings into backlog-ready items
- assign priority (`P0`/`P1`/`P2`) and effort (`S`/`M`/`L`) as best effort estimates
- ensure each backlog item maps 1:1 to a finding

Guidelines:
- Avoid combining unrelated changes into one item.
- Prefer small, independently deliverable improvements.
- Cross-role items must identify impacted roles or explicitly state “impact surface unknown.”

---

### 5) Generate Diagrams (arch-after.mmd) When Warranted
For each selected lens:
- Generate `arch-after.mmd` when warranted by findings, per `skills/reporting.md`.
- Do not generate diagrams for purely behavioral/config-only findings.

Diagrams must:
- remain within a single deployment unit
- avoid regions/cells/topology assumptions
- conform to Mermaid rules in `skills/reporting.md`

---

### 6) Generate Dependency-First Rollup Table (Mechanical Merge)
Generate `resilience/role_dependency_table.csv` as defined in `skills/reporting.md`.

Rules:
- Mechanical join only (no new findings)
- Coverage guarantee: every discovered dependency appears at least once
- Use structured join keys from `backlog.csv` (`DependencyIds`, `EntrypointIds`, `EndpointBindings`)

---

## Outputs

### Output Structure
All outputs are written under `resilience/` using lens-scoped directories plus the per-role mechanical rollup.

resilience/
├── discovery/
│   ├── discovery.json
│   └── arch-before.mmd
├── lens_plane-separation/
├── lens_domain-decoupling/
├── lens_modulith/
├── lens_dependency-isolation/
└── role_dependency_table.csv

### Per-lens Outputs (always)
Write lens-scoped artifacts under:
- `resilience/lens_<lens_name>/`

Each lens folder MUST include:
1) `role_assessment.md`
2) `backlog.csv`

Optional per lens (generated when warranted by findings):
3) `dependency_inventory.csv`
4) `arch-after.mmd`

Notes:
- Per-lens outputs must stand on their own.
- If a lens yields no meaningful findings, still emit `role_assessment.md` with “No findings” and list any open questions.

---

## Evidence Rules (Non-Negotiable)
- No findings without evidence.
- No speculative claims; use `UNKNOWN` + validation question when needed.
- Confidence must reflect evidence quality.
- Severity must be justified by customer impact and failure mechanics, not intuition.

---

## Phase Boundaries
This command:
- Produces Phase 1 outputs only
- Does not propose topology changes (multi-region, active-active, new deployables)
- Does not mandate microservices
- Produces inputs suitable for Phase 2/3 decisions (topology and regionalization), without asserting them