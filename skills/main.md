# Skill: System Fingerprinting & Orchestration

## Purpose
Identify the language, framework, and operational entrypoints of a repository in order to scope discovery and resiliency analysis correctly.

This skill establishes factual execution context and provides shared inputs to:
- `/resilience-map` (discovery)
- `/resilience-analyze` (lens-based analysis)

All downstream analysis must rely on the outputs of this skill rather than inferring or guessing.

---

## Scope

This skill is responsible for:
- Language and framework detection
- Identification of synchronous vs asynchronous execution entrypoints
- Producing normalized, evidence-backed entrypoint metadata for downstream skills

Non-goals:
- Proposing resiliency improvements
- Applying architectural lenses
- Inferring deployment topology, regions, or infrastructure characteristics

---

## Instructions

### 1) Language & Framework Detection

Detect the primary runtime language and framework using repository signals.

#### Java
- Look for: `pom.xml`, `build.gradle`, `settings.gradle`
- Identify framework when possible:
    - Spring Boot
    - Quarkus
    - Jakarta EE / JAX-RS
- If multiple modules exist, identify the primary deployable when metadata is present; otherwise enumerate and mark ambiguity.

#### Go
- Look for: `go.mod`
- Identify framework when possible:
    - `net/http`
    - Gin
    - Echo

#### Python
- Look for: `requirements.txt`, `pyproject.toml`
- Identify framework when possible:
    - FastAPI
    - Flask
    - Django

If multiple languages are present:
- Identify the primary runtime
- Mark others as secondary
- Emit an ambiguity note if ownership or role scope is unclear

---

### 2) Entrypoint Mapping

Identify inbound execution entrypoints and classify them by execution model.

This step focuses on **operational execution paths**.
Classification of control-plane vs operational behavior is handled later during discovery.

#### Synchronous Entrypoints (`type: sync`)

- Java:
    - `@RestController`
    - `@Controller`
    - `@RequestMapping`
    - JAX-RS resources
    - gRPC services (`BindableService`)
- Go:
    - `http.HandleFunc`
    - Router registrations
- Python:
    - Flask route decorators
    - FastAPI route decorators

#### Asynchronous Entrypoints (`type: async`)

- Java:
    - `@KafkaListener`
    - SQS consumers
    - `@Scheduled` jobs
    - Background workers
- Go:
    - Kafka / SQS consumers
    - Background goroutines acting as workers
- Python:
    - Celery tasks
    - Background consumers or workers

For each entrypoint, capture:
- id (stable; derived from name + file path)
- name (function / class / handler)
- type: `sync` | `async`
- framework
- route / topic / queue (if detectable)
- evidence: file path + line range

Entrypoints identified here are authoritative.
Additional entrypoints may only be added later with explicit evidence.

---

### 3) Output & Context Sharing

Store detected information as shared context for downstream skills, including:
- detected primary language and framework
- secondary languages (if any)
- list of entrypoints with sync/async classification
- evidence for each detection
- ambiguity notes or open questions (if applicable)

This context must be consumed by:
- `skills/discovery.md`
- `skills/lenses.md`
- `skills/reporting.md`

Downstream confidence must reflect the confidence of this detection step.

---

## Evidence Rules (Non-Negotiable)

- Do not infer language, framework, or entrypoints without evidence.
- If detection is ambiguous, mark as `unknown` and emit a clarification note.
- Do not infer:
    - deployment topology
    - resiliency behavior
    - SLAs or SLOs
- This output must be suitable for direct review by an engineer.