# Skill: System Fingerprinting & Orchestration

## Purpose
Identify the language, framework, and entry points of a repository in order to scope discovery and analysis correctly.

This skill provides shared context to:
- `/resilience-map` (discovery)
- `/resilience-analyze` (lens-based analysis)

All downstream analysis must rely on the outputs of this skill rather than guessing.

---

## Scope
This skill is responsible for:
- Language and framework detection
- Synchronous vs asynchronous entrypoint identification
- Providing normalized entrypoint metadata to other skills

Non-goals:
- Proposing resiliency improvements
- Applying architectural lenses
- Inferring topology or deployment characteristics

---

## Instructions

### 1) Language & Framework Detection
Detect the primary language and framework using repository signals.

**Java**
- Look for: `pom.xml`, `build.gradle`
- Identify framework when possible:
    - Spring Boot
    - Quarkus
    - Jakarta EE

**Go**
- Look for: `go.mod`
- Identify framework when possible:
    - `net/http`
    - Gin
    - Echo

**Python**
- Look for: `requirements.txt`, `pyproject.toml`
- Identify framework when possible:
    - FastAPI
    - Flask
    - Django

If multiple languages are present, identify the primary runtime and mark others as secondary.

---

### 2) Entry Point Mapping
Identify inbound execution entrypoints and classify them by execution model.

**Synchronous entrypoints**
- Java: `@RestController`, `@Controller`, JAX-RS resources
- Go: `http.HandleFunc`, router registrations
- Python: `@app.route`, FastAPI route decorators

**Asynchronous entrypoints**
- Java: `@KafkaListener`, SQS consumers, scheduled jobs
- Go: Kafka/SQS consumers, background workers
- Python: Celery tasks, background consumers

For each entrypoint, capture:
- entrypoint name / function / class
- type: synchronous or asynchronous
- framework
- evidence: file path + line range

---

### 3) Output & Context Sharing
Store detected information as shared context for downstream skills, including:
- detected language and framework
- list of operational entrypoints
- sync vs async classification
- evidence for each detection

This context must be consumed by:
- `skills/discovery.md`
- `skills/lenses.md`
- `skills/reporting.md`

---

## Evidence Rules
- Do not infer language, framework, or entrypoints without evidence.
- If detection is ambiguous, mark as `unknown` and emit a clarification question.
- Confidence in downstream findings must reflect the confidence of this detection.