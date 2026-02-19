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
- operational entrypoints (`sync` / `async`) with evidence

Entrypoints identified by `skills/main.md` are authoritative and MUST be reused.
Do not re-detect entrypoints unless additional ones are found with evidence.

Ensure each entrypoint includes:
- `id` (stable)
- `binding` (normalized when detectable; else `unknown`)
- `evidence`

---

### 2) Identify Control-Plane Operations (Best Effort)
Discover control-plane code paths, including:
- config / policy fetch or refresh
- feature flag reads
- entitlement / account / billing checks
- token or credential refresh
- background or scheduled refresh loops

Record invocation context when determinable; otherwise mark as `unknown`.

---

### 3) Identify Outbound Dependencies (Observed)
Identify outbound dependencies referenced in the repository:
- internal services
- third-party APIs
- datastores
- queues or streams

For each dependency, record:
- `dependency_id` (stable canonical slug)
- `dependency_name` (display name)
- `dependency_type`
- call sites with evidence
- call path classification (best effort): `sync_operational` | `control_plane` | `async` | `unknown`

For datastore/stream dependencies:
- Attempt to capture concrete identifiers (e.g., table names, topic names, queue names, bucket names) **only when visible** in code or config.
- Otherwise mark as `unknown`.

Do not infer behavior, guarantees, SLAs, or failure posture.

---

### 4) Populate Minimal Regionalization Facts (Optional, Evidence-Backed)
Populate `regionalization_facts.data_assets_observed[]` as defined in `skills/discovery.md`:
- asset_type: `table` | `stream` | `cache` | `bucket` | `unknown`
- engine: `dynamodb` | `redis` | `kafka` | `sqs` | `s3` | `unknown`
- identifier: only if evidenced; else `unknown`
- used_on_paths[]: `sync_operational` | `control_plane` | `async` | `unknown`
- evidence: file path + line range

Rules:
- This step is observational only.
- Do not infer locality, replication, or residency posture.

---

## Outputs
Write all outputs under a `resilience/` directory.

### Required outputs
1) `resilience/discovery/discovery.json`
    - Must conform to the model defined in `skills/discovery.md`.

2) `resilience/discovery/arch-before.mmd`
    - Mermaid diagram representing the current state (facts only):
        - role boundary
        - operational entrypoints
        - control-plane operations (if identifiable)
        - external dependencies
    - Must not include proposed changes.
    - Must conform to Mermaid generation rules in `skills/reporting.md`.

---

## Evidence Rules (Non-Negotiable)
- No facts without evidence.
- Unknowns must be explicitly labeled and added to `open_questions[]`.
- No speculation, inferred architecture, or inferred topology.

---

## Output Quality Bar
This command is successful when:
- An engineer can locate every entrypoint and dependency using the provided evidence.
- `arch-before.mmd` is consistent with `discovery.json` (no contradictions).