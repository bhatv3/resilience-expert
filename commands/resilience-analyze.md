# Command: /resilience-analyze

## Intent
Analyze a role/service repository using the Architectural Lenses and produce Phase 1 resiliency artifacts that are directly consumable for prioritization and execution.

This command must be evidence-backed, Phase 1 scoped, and must not assume topology changes.

---

## Inputs
- One or more repositories (local workspace).
- Optional: an existing `resilience/discovery.json` produced by `/resilience-map`.
    - If not present, run the equivalent discovery steps inline (best effort), limited to the scope needed for the selected lenses, and mark unknowns explicitly.

Optional parameters (best effort if supported):
- `lens`: one of `{control-plane, domain-decoupling, modulith, dependency-isolation, all}` (default: `all`)
- `phase`: `1` (default and only supported)
- `flows`: N critical entrypoints to focus on (default: `3`)

---

## Execution Steps (Do These In Order)

### 1) Load or Generate Discovery Map
If `resilience/discovery.json` exists, load it.
Otherwise, generate a best-effort discovery map that captures at minimum:
- operational entrypoints
- control-plane operations (best effort)
- dependencies and call sites (best effort)

Notes:
- Do not force “virtual module” inference during inline discovery.
- All unknowns must be explicitly labeled and surfaced as validation questions.

### 2) Select Critical Entrypoints / Flows
Select up to N critical operational entrypoints to anchor analysis:
- prefer explicitly configured "critical flows" if present
- otherwise choose by best-effort heuristics:
    - centrality (high fan-out)
    - business importance (send/check verification, fallback, callbacks/consumers)
    - frequency indicators (if detectable)

Record:
- selected entrypoints (with evidence)
- selection rationale

### 3) Apply Lenses
For each selected lens, produce findings that conform to the lens contracts in `skills/lenses.md` and the rigor rules in `skills/verification.md`.

Supported lenses:
- Lens 1: Control Plane vs Operational Plane Separation
- Lens 2: Domain Decoupling via Explicit Interfaces
- Lens 3: Monolith → Modulith (Lightweight, Improvement-Driven)
- Lens 4: Dependency Criticality & Isolation

Each finding MUST include:
- severity (`P0`/`P1`/`P2`)
- confidence (`High`/`Medium`/`Low`)
- evidence (file + line range or config key + location)
- impacted entrypoints (if applicable)
- why customers care (1 sentence)
- recommended change (1–3 bullets) OR investigation task (if evidence is insufficient)
- validation (1–3 bullets)

If evidence cannot be found:
- mark as `UNKNOWN`
- emit a validation question instead of a recommendation

### 4) Consolidate and Prioritize Improvements
Combine findings into a set of actionable improvements and assign:
- priority (`P0`/`P1`/`P2`) using the rubric in `skills/reporting.md`
- effort (`S`/`M`/`L`) as a best-effort estimate
- expected customer impact
- validation plan

Guidelines:
- Avoid combining unrelated changes into one item.
- Prefer small, independently deliverable improvements.
- Cross-role items must identify impacted roles or explicitly state “impact surface unknown.”

---

## Outputs
Write all outputs under a `resilience/` directory.

Required artifacts (Phase 1 bundle):
1) `resilience/role_assessment.md`
2) `resilience/backlog.csv`
3) `resilience/dependency_inventory.csv`
    - Catalog of dependencies referenced by findings, including call path and criticality classification (best effort).
4) `resilience/arch-before.mmd`
5) `resilience/arch-after.mmd` (optional; only when Phase 1 internal boundary changes are identified)

Notes:
- `arch-before.mmd` must reflect discovery facts only.
- `arch-after.mmd` must not introduce new regions or deployment units.
- If no meaningful internal boundary changes are identified, do not generate `arch-after.mmd`.

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
- Does not propose topology changes (multi-region, active-active)
- Does not mandate microservices
- Produces inputs for Phase 2/3 decisions (topology and regionalization)