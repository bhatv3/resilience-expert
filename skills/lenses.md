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
Prevent configuration, policy, or management logic from destabilizing live verification traffic.

### Risk Patterns to Detect
- Control-plane operations executed synchronously on request paths
- Repeated config or policy fetches per request
- Control-plane dependency failures that block operational flows
- Shared thread pools between control and operational work

### Evidence to Look For
- Control operations invoked inside request handlers
- Absence of caching or last-known-good usage
- No timeouts or fail-open behavior for control reads

### Improvement Candidates (Examples)
- Local caching with TTL for control data
- Async refresh of configuration
- Fail-open defaults for non-critical control reads
- Separate executors / thread pools

### Outcome Enabled
Operational traffic remains stable and predictable even when control systems are degraded.

---

## Lens 2: Domain Decoupling via Explicit Interfaces

### Goal
Prevent Verify reliability from depending on undocumented or implicit behavior of external domains.

### Risk Patterns to Detect
- Direct use of external domain models in core logic
- Business decisions coupled to specific error semantics
- Multiple call sites to the same external service without a shared adapter

### Evidence to Look For
- External SDK types crossing service boundaries
- Error handling logic tied to specific upstream responses
- No abstraction layer around external calls

### Improvement Candidates (Examples)
- Introduce explicit adapters / facades
- Normalize external errors into Verify-owned semantics
- Centralize retry / timeout / fallback logic

### Outcome Enabled
External incidents do not cascade by default, and Verify can evolve independently.

---

## Lens 3: Monolith → Modulith (Lightweight, Improvement-Driven)

### Goal
Create durable internal boundaries that limit failure propagation without introducing microservices.

### Risk Patterns to Detect
- Cross-cutting concerns tightly interwoven (routing, fraud, messaging)
- Shared thread pools or resource contention between unrelated subsystems
- Broad, implicit dependencies across large code areas

### Evidence to Look For
- Call graphs showing unrelated flows sharing execution paths
- Common utility classes acting as implicit integration points
- Lack of clear ownership boundaries in code

### Improvement Candidates (Examples)
- Introduce internal facades between subsystems
- Separate executors for high-risk flows
- Establish internal APIs with clear ownership

### Outcome Enabled
Failures are isolated along intentional internal boundaries, enabling incremental evolution.

---

## Lens 4: Dependency Criticality and Isolation

### Goal
Reduce cascading failures and thread exhaustion caused by slow or failing dependencies.

### Risk Patterns to Detect
- Missing timeouts or unbounded retries
- Synchronous calls to non-critical dependencies
- Shared thread pools for all outbound calls
- No local caching for read-heavy dependencies

### Evidence to Look For
- Client instantiation without timeouts
- Retry loops without backoff or limits
- Blocking calls on request threads
- Repeated remote reads of stable data

### Improvement Candidates (Examples)
- Timeouts + circuit breakers
- Bulkheads / isolated thread pools
- Local caches or last-known-good snapshots
- Async or deferred dependency access

### Outcome Enabled
Verify degrades gracefully under dependency failure instead of amplifying outages.

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