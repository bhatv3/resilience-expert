# Skill: Resiliency Analysis Lenses

## Purpose
Apply a consistent set of analytical lenses to discovery outputs in order to identify concrete, Phase-1 resiliency improvements.

Each lens:
- identifies specific risk patterns
- explains why they matter
- proposes improvement candidates scoped to the existing deployment topology

Lenses must produce findings backed by evidence and framed for execution, not theory.

---

## Lens 1: Control Plane vs Operational Plane Separation

### Goal
Prevent configuration, policy, and control enforcement logic from destabilizing live verification traffic.

---

### Risk Patterns to Detect

**Synchronous control work on the operational path**
- Control-plane operations executed synchronously on request paths
- Repeated config, policy, or entitlement fetches per request
- Control-plane dependency failures that block operational flows

**Shared persistence and capacity coupling**
- Control-plane and operational-plane reads/writes sharing:
    - the same database tables or collections
    - the same partition keys or hot key space
    - the same capacity pools (e.g., DynamoDB WCUs/RCUs, shared Redis clusters)
- Bulk or administrative control-plane operations contending with live traffic

**Control-like enforcement on the operational path**
- Rate limiting, quotas, or entitlement checks implemented as:
    - synchronous database reads/writes per request
    - multiple sequential checks on a single request path
- Fail-closed behavior when rate-limit or policy stores are unavailable

**Resource coupling**
- Shared thread pools or executors between control-plane and operational work
- Control-plane cache refresh or backfill work executing on request threads

---

### Evidence to Look For

- Control operations invoked inside request handlers or core execution paths
- Rate limit / quota / entitlement checks implemented as synchronous calls
- Writes to shared tables or counters during request execution
- Absence of:
    - local caching or last-known-good snapshots
    - explicit timeouts, circuit breakers, or fail-open defaults
- Shared clients, connection pools, or executors used by both planes

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
Operational verification traffic remains stable and predictable even when configuration systems, policy enforcement, rate limiting stores, or other control-plane components are degraded or unavailable.

---

## Lens 2: Domain Decoupling via Explicit Interfaces

### Goal
Prevent Verify reliability from depending on undocumented, implicit, or behavior-specific assumptions about external domains.

---

### Risk Patterns to Detect

**Code-level coupling**
- Direct use of external domain models or SDK types in core logic
- External errors or responses passed through without normalization
- Multiple call sites to the same external service without a shared adapter

**Behavioral and semantic coupling**
- Business decisions tied to specific upstream error codes or message formats
- Assumptions about upstream delivery guarantees, ordering, or retries
- Logic that implicitly relies on “eventual callbacks” or side channels

**Change and lifecycle coupling**
- Verify behavior changes when upstream domains deploy or reconfigure
- Tight coordination required for safe rollout or rollback across domains
- Lack of versioned contracts or compatibility guarantees

**Control-plane leakage across domains**
- Reliance on internal-only APIs, Kafka topics, or callbacks without formal contracts
- Verify using upstream internals rather than supported, versioned interfaces

---

### Evidence to Look For

- External SDK or API response types crossing into core decision logic
- Error-handling branches keyed to specific upstream responses
- Duplicate integration logic spread across multiple code paths
- References to internal topics, callbacks, or undocumented APIs
- Comments or code indicating assumptions about upstream behavior (“this will always arrive”)

---

### Improvement Candidates (Examples)

**Interface hardening**
- Introduce explicit adapters or facades per external domain
- Normalize external responses into Verify-owned models and error semantics
- Version adapters to allow independent evolution

**Behavioral decoupling**
- Make delivery, status, and callback guarantees explicit and defensive
- Design for missing, delayed, or duplicated upstream signals
- Treat upstream signals as advisory, not authoritative, where possible

**Change isolation**
- Centralize integration logic to reduce blast radius of upstream changes
- Add feature flags or compatibility layers around external behavior changes
- Prefer pull-based or state-based reconciliation over event-only reliance

---

### Outcome Enabled
External domain incidents, behavior changes, or delivery anomalies do not automatically cascade into Verify outages, and Verify can evolve, deploy, and recover independently.

---

## Lens 3: Monolith → Modulith (Lightweight, Improvement-Driven)

### Goal
Create durable internal boundaries that limit failure propagation and reduce change-coupling, while continuing to deploy as a single process.

A “boundary” in Phase 1 means: a clear internal seam (facade/adapters + ownership + resource isolation) that makes propagation and coupling visible and controllable, without requiring new deployables.

---

### Risk Patterns to Detect

**Cross-cutting concerns and change-coupling**
- Cross-cutting concerns tightly interwoven (routing, fraud, messaging, policy)
- Many unrelated flows depending on the same “core” classes or utilities
- Widespread conditionals / feature flags controlling behavior across the codebase

**Hidden propagation paths**
- Shared thread pools, executors, or queues used across unrelated subsystems
- Shared caches or shared in-memory state used by multiple concerns
- Shared persistence tables/collections used as an integration mechanism between subsystems

**Structural coupling**
- Broad imports across packages without a small set of “public” APIs
- Direct calls into implementation packages (no stable seams)
- Cyclic dependencies between major subsystems (A → B → A)

---

### Evidence to Look For
- Call graphs or cross-references showing subsystems sharing execution paths
- High fan-in “hub” classes used by multiple concerns (coupling hotspots)
- Imports/references crossing subsystem boundaries (especially into internal/impl packages)
- Cyclic dependency signals (package A references B and vice versa)
- Shared resource usage:
    - same executor/thread pool
    - shared cache instance
    - same persistence table used by multiple concerns

---

### Improvement Candidates (Examples)

**Introduce seams (without new deployables)**
- Create internal facades between subsystems (routing, fraud, messaging, policy)
- Adopt an “ports/adapters” structure for external integrations
- Define stable internal APIs and move direct implementation calls behind them

**Isolate failure domains inside the process**
- Separate executors / thread pools for high-risk subsystems
- Bound queue sizes and apply backpressure per subsystem
- Isolate caches and shared state (avoid global mutable singletons)

**Reduce structural coupling**
- Consolidate shared utilities into explicit shared libraries (or narrow helper APIs)
- Remove cycles by introducing dependency inversion (interfaces at the boundary)

---

### Outcome Enabled
Failures and overload are contained within intentional internal boundaries, enabling incremental resiliency improvements and future topology evolution along validated fault lines rather than forcing new ones.

---

## Lens 4: Dependency Criticality and Isolation

### Goal
Reduce cascading failures, retry amplification, and thread/resource exhaustion caused by slow or failing dependencies.

---

### Risk Patterns to Detect

**Timeout / retry posture gaps**
- Missing timeouts or overly large timeout budgets on outbound calls
- Unbounded retries or retries without backoff/jitter
- Retries applied uniformly (including on non-retryable errors)

**Retry amplification / thundering herds**
- Fanout calls multiplied by retries (N calls × R retries)
- Aggressive client-side retries during partial outages
- Lack of request hedging controls or admission control under overload

**Critical-path coupling**
- Synchronous calls to non-critical dependencies on the request path
- Multiple sequential dependency calls on the critical path (latency stacking)
- Dependency failure resulting in fail-closed behavior without explicit intent

**Resource contention / exhaustion**
- Shared thread pools or executors for all outbound calls
- Blocking calls on request threads without bulkheads
- Unbounded queues or work submission during dependency slowdown

**Read-path inefficiency**
- Repeated remote reads of stable data (config, entitlements, routing hints)
- No caching or last-known-good snapshots for high-read dependencies

**Correctness risks under retries**
- Retries performed without idempotency guarantees
- Duplicate side effects (double writes, double sends) not prevented or reconciled
- Missing request correlation IDs / idempotency keys for downstream calls

---

### Evidence to Look For
- Client instantiation without explicit timeouts (connect + read)
- Retry loops without max attempts, backoff, or jitter
- Retry policies applied to broad exception classes
- Blocking calls on request threads and shared executors
- Lack of bulkhead isolation per dependency or dependency class
- High fanout patterns (multiple dependency calls per request path)
- Lack of idempotency keys / dedupe mechanisms for side-effecting calls
- Repeated remote reads of stable data without caching/LKG

---

### Dependency Criticality Classification (Lightweight)
Classify each dependency on the operational path as one of:
- **Hard-critical:** request cannot succeed without it (must be reliable and isolated)
- **Soft-critical:** request can succeed with degraded behavior (must have explicit fallback/degrade)
- **Non-critical:** should never block the request (async, best-effort, or fail-open)

This classification should be evidence-backed and tied to observed behavior in code.

---

### Improvement Candidates (Examples)

**Isolation and fast-fail**
- Tight, explicit timeouts aligned to budgets
- Circuit breakers with sensible half-open recovery
- Bulkheads / isolated thread pools per critical dependency class

**Retry discipline**
- Bounded retries with exponential backoff + jitter
- Retry only on explicitly retryable failure modes
- Admission control or load shedding under dependency degradation

**Degradation and caching**
- Local caches / last-known-good snapshots for remote reads
- Async or deferred access for soft/non-critical dependencies
- Explicit fail-open / degrade behavior for soft-critical dependencies

**Correctness hardening**
- Idempotency keys for side-effecting calls
- Dedupe and reconciliation for “at least once” behaviors
- Correlation IDs and structured logging for dependency calls

---

### Outcome Enabled
Verify degrades gracefully under dependency failure instead of amplifying outages, while maintaining correctness under retries and partial failure.

---

## Lens Output Requirements
Each lens must emit findings that include:
- description of the issue
- evidence (file + line range)
- customer impact
- severity (P0/P1/P2)
- confidence (High/Medium/Low)
- recommended improvement
- validation notes (if needed)

---