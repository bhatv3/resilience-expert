# Skill: Resiliency Analysis Lenses (Phase 1)

## Purpose
Apply a consistent set of analytical lenses to discovery outputs in order to identify **concrete, Phase-1 resiliency improvements**.

Each lens:
- targets specific, observable risk patterns
- explains why those risks matter
- proposes improvement candidates scoped to the existing deployment topology

Lenses are designed to produce **execution-ready findings**, not theoretical architecture guidance.

---

## General Lens Rules
All lenses must:
- operate only on evidence produced by discovery
- comply with rigor rules defined in `skills/verification.md`
- produce findings scoped to a single role / deployment unit
- avoid assuming topology, regions, replication, or infrastructure changes

Lenses **identify problems and improvement candidates**; they do not mandate designs or future states.

---

## Lens 1: Plane Separation (Control vs Operational)

### Goal
Prevent configuration, policy, and control enforcement logic from destabilizing live verification traffic.

---

### Risk Patterns to Detect

#### Synchronous control work on the operational path
- Control-plane operations executed synchronously on request paths
- Repeated config, policy, or entitlement fetches per request
- Control-plane dependency failures that block operational flows

#### Shared persistence and capacity coupling
- Control-plane and operational-plane reads/writes sharing:
    - the same database tables or collections
    - the same partition keys or hot-key space
    - the same capacity pools (e.g., DynamoDB RCUs/WCUs, shared Redis clusters)
- Bulk or administrative control-plane operations contending with live traffic

#### Control-like enforcement on the hot path
- Rate limiting, quotas, or entitlement checks implemented as:
    - synchronous database reads/writes per request
    - multiple sequential checks on a single request path
- Fail-closed behavior when rate-limit or policy stores are unavailable

#### Resource coupling
- Shared thread pools or executors between control-plane and operational work
- Control-plane refresh, backfill, or reconciliation work executing on request threads

---

### Evidence to Look For
- Control operations invoked inside request handlers or core execution paths
- Synchronous rate-limit / quota / entitlement checks
- Writes to shared tables or counters during request execution
- Absence of:
    - local caching or last-known-good snapshots
    - explicit timeouts, circuit breakers, or degrade behavior
- Shared clients, connection pools, or executors used by both planes
- Evidence that control-plane and operational-plane code paths reference the same datastore identifiers

---

### Improvement Candidates (Examples)

**Control data isolation**
- Local caching with TTL for configuration, policy, and entitlement data
- Last-known-good reads for control data under failure
- Async or background refresh of control state

**Persistence and capacity isolation**
- Separate tables or keyspaces for control-plane vs operational data
- Separate capacity allocation or autoscaling policies
- Separate database clients or connection pools

**Rate limiting and enforcement hardening**
- In-memory or local token-bucket rate limiters with bounded drift
- Async counter aggregation instead of synchronous writes
- Explicit degradation behavior (fail-open, soft limits) for low-risk flows

**Execution isolation**
- Separate executors or thread pools for control-plane work
- Single-flight cache refresh and jittered expiration to avoid stampedes

---

### Outcome Enabled
Operational verification traffic remains stable and predictable even when configuration systems, policy enforcement, or rate-limiting components are degraded or unavailable.

---

## Lens 2: Domain Decoupling via Explicit Interfaces

### Goal
Prevent Verify reliability from depending on undocumented, implicit, or behavior-specific assumptions about external domains.

---

### Risk Patterns to Detect

#### Code-level coupling
- Direct use of external domain models or SDK types in core logic
- External errors or responses passed through without normalization
- Multiple call sites to the same external service without a shared adapter

#### Behavioral and semantic coupling
- Business decisions tied to specific upstream error codes or message formats
- Assumptions about upstream delivery guarantees, ordering, or retries
- Logic that implicitly relies on “eventual callbacks” or side channels

#### Change and lifecycle coupling
- Verify behavior changing when upstream domains deploy or reconfigure
- Tight coordination required for safe rollout or rollback across domains
- Lack of versioned contracts or compatibility guarantees

#### Control-plane leakage across domains
- Reliance on internal-only APIs, Kafka topics, or callbacks without formal contracts
- Use of upstream internals rather than supported, versioned interfaces

---

### Evidence to Look For
- External SDK or API response types crossing into core decision logic
- Error-handling branches keyed to specific upstream responses
- Duplicate integration logic spread across multiple code paths
- References to internal topics, callbacks, or undocumented APIs
- Code comments indicating assumptions about upstream behavior

---

### Improvement Candidates (Examples)

**Interface hardening**
- Introduce explicit adapters or facades per external domain
- Normalize external responses into Verify-owned models and error semantics
- Version adapters to allow independent evolution

**Behavioral decoupling**
- Make delivery, status, and callback guarantees explicit and defensive
- Design for missing, delayed, or duplicated upstream signals
- Treat upstream signals as advisory where appropriate

**Change isolation**
- Centralize integration logic to reduce blast radius
- Add compatibility layers or feature flags around external behavior
- Prefer reconciliation/state-based models over event-only reliance

---

### Outcome Enabled
External domain incidents or behavior changes do not automatically cascade into Verify outages, and Verify can evolve, deploy, and recover independently.

---

## Lens 3: Monolith → Modulith (Lightweight, Improvement-Driven)

### Goal
Create durable internal boundaries that limit failure propagation and reduce change-coupling while continuing to deploy as a single process.

A Phase-1 boundary is a **clear internal seam** (facade + ownership + resource isolation), not a new deployable.

---

### Risk Patterns to Detect

#### Cross-cutting concerns and change-coupling
- Routing, fraud, messaging, policy, and core logic tightly interwoven
- Many unrelated flows depending on the same core classes or utilities
- Widespread conditionals controlling behavior across subsystems

#### Hidden propagation paths
- Shared thread pools, executors, or queues across unrelated subsystems
- Shared caches or in-memory state used by multiple concerns
- Shared persistence tables used as implicit integration points

#### Structural coupling
- Broad imports across packages without stable public APIs
- Direct calls into implementation packages
- Cyclic dependencies between major subsystems

---

### Evidence to Look For
- Call graphs showing unrelated flows sharing execution paths
- High fan-in “hub” classes used by multiple subsystems
- Imports crossing subsystem boundaries into internal packages
- Cyclic package dependencies
- Shared executors, caches, or persistence identifiers

---

### Improvement Candidates (Examples)

**Introduce seams (without new deployables)**
- Internal facades between routing, fraud, messaging, and policy
- Ports/adapters structure for integrations
- Stable internal APIs with clear ownership

**Isolate failure domains**
- Separate executors or thread pools for high-risk subsystems
- Bounded queues and backpressure per subsystem
- Avoid global mutable state and shared caches

**Reduce structural coupling**
- Narrow shared utilities into explicit helper APIs
- Remove cycles via dependency inversion

---

### Outcome Enabled
Failures and overload are contained within intentional internal boundaries, enabling incremental resiliency gains and future topology evolution along validated fault lines.

---

## Lens 4: Dependency Criticality and Isolation

### Goal
Reduce cascading failures, retry amplification, and resource exhaustion caused by slow or failing dependencies.

---

### Risk Patterns to Detect

#### Timeout and retry posture gaps
- Missing or excessive timeouts
- Unbounded retries or retries without backoff/jitter
- Retry applied indiscriminately across error classes

#### Retry amplification
- Fan-out calls multiplied by retries
- Aggressive retries during partial outages
- No admission control or load shedding

#### Critical-path coupling
- Synchronous calls to non-critical dependencies
- Sequential dependency calls stacking latency
- Fail-closed behavior without explicit intent

#### Resource exhaustion
- Shared executors for all outbound calls
- Blocking calls on request threads
- Unbounded work queues during slowdown

#### Correctness under retries
- Retries without idempotency guarantees
- Duplicate side effects not prevented or reconciled
- Missing idempotency or correlation identifiers

---

### Dependency Criticality Classification
Classify each dependency on the operational path as:
- **Hard-critical** — request cannot succeed without it
- **Soft-critical** — request can succeed in degraded mode
- **Non-critical** — should never block the request

Classification must be evidence-backed and tied to observed behavior.

---

### Improvement Candidates (Examples)

**Isolation and fast-fail**
- Tight, explicit timeouts
- Circuit breakers with recovery semantics
- Bulkheads per dependency class

**Retry discipline**
- Bounded retries with backoff and jitter
- Retry only explicitly retryable failures
- Admission control under degradation

**Degradation and caching**
- Local caches or last-known-good snapshots
- Async or deferred access for soft/non-critical dependencies
- Explicit fail-open behavior where safe

**Correctness hardening**
- Idempotency keys for side-effecting calls
- Dedupe and reconciliation mechanisms
- Correlation IDs and structured logging

---

### Outcome Enabled
Verify degrades gracefully under dependency failure instead of amplifying outages, while maintaining correctness under retries and partial failure.

---

## Lens Output Requirements
Each lens must emit findings that include:
- description of the issue
- evidence (file + line range)
- customer impact
- severity (P0 / P1 / P2)
- confidence (High / Medium / Low)
- recommended improvement or investigation
- validation notes (if needed)