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

### 2) `entrypoints[]`
Operational execution entrypoints into the system.

For each entrypoint, capture:
- `name`
- `type`: `sync` | `async`
- `framework`
- `route` / `topic` / `queue` (if detectable)
- `evidence`: file path + line range

Entrypoints identified in `skills/main.md` are authoritative.
Additional entrypoints may be added only with evidence.

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

### 4) `dependencies[]`
Observed outbound dependencies referenced in the codebase.

For each dependency:
- `dependency_name`
- `dependency_type`:
    - `internal_service`
    - `third_party_api`
    - `datastore`
    - `queue_or_stream`
- `call_sites[]` with evidence
- `call_path` (best effort):
    - `sync_operational`
    - `async`
    - `control_plane`
    - `unknown`

Do not infer dependency semantics, guarantees, or SLAs.

---

### 5) `regionalization_facts` (best effort, evidence-backed)
A minimal, observational appendix to support global vs local classification without introducing analysis.

#### 5.1) `regionalization_facts.data_assets_observed[]`
For each observed data asset, capture only what is explicitly visible in code or configuration.

Each entry should include:
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