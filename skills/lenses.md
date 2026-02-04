# Skill: Architectural Lenses

## Lens 1: Control vs. Data Plane Separation
**Goal:** Prevent management logic from blocking business traffic.
**Detection Logic:**
1.  Search for "Control Operations": Config refreshes, Auth token rotation, Health checks, Metrics pushing.
2.  Search for "Data Operations": The Entry Points identified in Phase 1.
3.  **Violation:** If a Control Operation is called *synchronously* within a Data Operation without a timeout or fallback.
    * *Pattern:* `function handleRequest() { refreshConfig(); ... }`

## Lens 2: Explicit Contracts (Anti-Corruption Layer)
**Goal:** Prevent external domain models from leaking into core logic.
**Detection Logic:**
1.  Identify external libraries (Stripe, AWS SDK, Upstream gRPC clients).
2.  Trace the return objects of these libraries.
3.  **Violation:** If an external type (e.g., `com.stripe.model.Charge`) appears in:
    * The Database Layer (saved directly).
    * The Core Domain Logic (passed as a function argument).
    * The UI/API Response (returned directly to client).

## Lens 3: Monolith to Modulith
**Goal:** Enforce strict visibility.
**Detection Logic:**
1.  Using the "Virtual Modules" from Discovery:
2.  **Violation:** If `Module::A` imports a class from `Module::B`'s internal/impl package instead of its API package.
3.  **Violation:** Circular dependencies (`A -> B -> A`).