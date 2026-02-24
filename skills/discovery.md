# Skill: Discovery (Phase 1)

## Purpose
Produce a factual, evidence-backed discovery model of a repository to support Phase 1 resiliency analysis.

This skill consumes system fingerprinting and entrypoint information from `skills/main.md` and emits a normalized discovery artifact.
Discovery is **observational only** and must **not** propose improvements, risk assessments, or architectural changes.

---

## Inputs
From `skills/main.md`:
- detected language and framework
- operational entrypoints (sync / async) with evidence

From repository contents:
- source code
- configuration files
- build manifests

---

## Discovery Output Model
Discovery produces a single normalized model with the following sections.

### 1) `role_metadata` (best effort)
- `role_name` (from metadata file if present; else `unknown`)
- `repositories_analyzed`
- `owner_team` (if present; else `unknown`)

---

### 2) `operational_entrypoints[]`
Operational execution entrypoints into the system.

Entrypoints identified in `skills/main.md` are authoritative.
Additional entrypoints may be added only with evidence.

For each entrypoint, capture:
- `id` (stable; derived from name + file path)
- `name`
- `type`: `sync` | `async`
- `framework`
- `binding` (route / topic / queue / schedule, if detectable)
- `evidence`: file path + line range

Notes:
- `binding` should be the concrete route/topic/queue when directly detectable; otherwise `unknown`.
- Do not infer execution semantics beyond the `sync`/`async` classification from `skills/main.md`.

---

### 3) `control_plane_ops[]` (best effort)
Code paths that represent control-plane behavior, including:
- configuration or policy fetch / refresh
- feature flag reads
- entitlement / account / billing checks
- credential or token refresh
- scheduled or background refresh jobs

For each operation, capture:
- `name` / function
- `invocation_context` (request path vs background, if determinable)
- `evidence`

If invocation context is ambiguous, mark as `unknown`.

---

### 4) `outbound_dependencies[]`
Observed outbound dependencies referenced in the codebase.

Outbound dependencies include **services and data-plane systems** used by the role, such as:
- internal HTTP/gRPC services
- third-party APIs
- datastores (e.g., DynamoDB)
- caches (e.g., Redis)
- queues and streams (e.g., SQS, Kafka)

For each dependency:
- `dependency_id` (stable canonical slug; e.g., `accsec-verify-routing`, `dynamodb-comms`, `redis-feedback-cache`)
- `dependency_name` (human-readable display name)
- `dependency_type`:
    - `internal_service`
    - `third_party_api`
    - `datastore`
    - `cache`
    - `queue_or_stream`
- `call_sites[]` with evidence
- `call_path` (best effort):
    - `sync_operational`
    - `async`
    - `control_plane`
    - `unknown`
- `data_assets[]` (optional; best effort)
    - list of `asset_id` values referencing `regionalization_facts.data_assets_observed[]`

Rules:
- Do not infer dependency semantics, guarantees, or SLAs.
- `dependency_id` MUST be stable within the repo and consistent across lens outputs.
- If canonicalization is ambiguous, use a best-effort slug and add an open question.
- `data_assets[]` MUST reference only `asset_id`s that are explicitly emitted under `regionalization_facts.data_assets_observed[]`.
- If a dependency clearly maps to a data asset but the asset identifier is not visible (and thus the asset `identifier` is `unknown`), either:
    - omit `data_assets[]`, or
    - reference an asset whose `identifier` is `unknown` but still has an `asset_id` (only if evidence supports the existence of the asset).

---

### 5) `regionalization_facts` (best effort, evidence-backed)
A minimal, observational appendix to support global vs local classification without introducing analysis.

#### 5.1) `regionalization_facts.data_assets_observed[]`
For each observed data asset, capture only what is explicitly visible in code or configuration.

Each entry should include:
- `asset_id` (stable identifier for cross-referencing; derived from engine + identifier or evidence location; if ambiguous, use best-effort and add an open question)
- `asset_type`: `table` | `stream` | `cache` | `bucket` | `unknown`
- `engine`: `dynamodb` | `redis` | `kafka` | `sqs` | `s3` | `unknown`
- `identifier`: concrete name when visible (e.g., table name, topic, queue, bucket); else `unknown`
- `used_on_paths[]`: one or more of:
    - `sync_operational`
    - `control_plane`
    - `async`
    - `unknown`
- `evidence`: file path + line range

Rules:
- Only include identifiers that are directly evidenced.
- Do not infer regional locality, replication, tenancy, or data residency.
- `asset_id` exists to support deterministic linking from `outbound_dependencies[]` and must not encode inferred topology.

This section is optional; if no data assets can be observed with evidence, emit an empty list.

---

### 6) `open_questions[]`
Explicit list of ambiguities or missing evidence discovered during analysis.

Each question should include:
- `description`
- `related_files_or_areas`
- `why_clarification_matters`

---

## Discovery Rules (Non-Negotiable)
- No assertions without evidence.
- If classification is ambiguous, mark as `unknown`.
- Do not infer:
    - production topology
    - resiliency behavior
    - SLOs or SLAs
    - regional replication or data residency posture
- Discovery must be suitable for direct review by an engineer.