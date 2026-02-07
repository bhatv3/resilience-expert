# Command: /resilience-map

## Intent
Generate a factual, evidence-backed discovery map of a role/service repository.

This command performs discovery only. It does not apply resiliency lenses or propose improvements.
The output is designed to be consumed by `/resilience-analyze`.

---

## Inputs
- One or more repositories (local workspace).
- Optional repository metadata file (e.g., `resiliency-skill.yaml`) providing:
    - role name
    - owner team
    - critical user flows (optional)

If metadata is missing, proceed with discovery and mark fields as `unknown`.

---

## Execution Steps
Perform discovery using the normalized model defined in `skills/discovery.md`.

### 1) Load System Fingerprinting Context
Consume outputs from `skills/main.md`, including:
- detected language and framework
- operational entrypoints (sync / async) with evidence

Do not re-detect entrypoints unless additional ones are found with evidence.

---

### 2) Identify Control-Plane Operations
Discover control-plane code paths, including:
- config / policy fetch or refresh
- feature flag reads
- entitlement / account / billing checks
- token or credential refresh
- background or scheduled refresh loops

Record invocation context when determinable; otherwise mark as `unknown`.

---

### 3) Identify Outbound Dependencies
Identify outbound dependencies referenced in the repository:
- internal services
- third-party APIs
- datastores
- queues or streams

For each dependency, record:
- dependency name and type
- call sites with evidence
- best effort call-path classification

For datastore dependencies, attempt to capture concrete identifiers (e.g., table names, schemas, cluster endpoints) when they are visible in code or configuration; otherwise mark as `unknown`.

Do not infer behavior or reliability characteristics.

---

## Outputs
Write all outputs under a `resilience/` directory.

### Required outputs
1) `resilience/discovery/discovery.json`
   A structured JSON document conforming to the model defined in `skills/discovery.md`.

2) `resilience/discovery/arch-before.mmd`
   A Mermaid diagram representing the current state:
- role boundary
- operational entrypoints
- control-plane operations (if identifiable)
- external dependencies

The diagram must reflect discovered facts only.

---

## Evidence Rules (Non-Negotiable)
- No facts without evidence.
- Unknowns must be explicitly labeled and added to `open_questions`.
- No speculation or inferred architecture.

---

## Output Quality Bar
This command is successful when:
- An engineer can locate every entrypoint and dependency using the provided evidence.
- The diagram matches the discovery.json contents with no contradictions.