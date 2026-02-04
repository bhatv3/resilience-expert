# Skill: Artifact Generation

## Objective
Convert analysis data into standardized files.

## Task 1: Generate `resilience/impact_map.json`
Output a JSON file containing all verified findings:
```json
{
  "timestamp": "...",
  "modules_scanned": ["..."],
  "findings": [
    {
      "type": "ControlPlaneViolation",
      "severity": "High",
      "file": "src/PaymentService.java",
      "line": 45,
      "description": "Blocking config call inside payment flow.",
      "impacted_endpoint": "POST /pay"
    }
  ]
}
```

## Task 2: Generate `resilience/architecture.mermaid`
Create a Mermaid diagram file visualizing the system health. Rules:

Nodes: Represent the "Virtual Modules" and "External Systems" (DBs, APIs).

Edges: Represent calls/dependencies found in the code.

Styling:

Use classDef failure fill:#ffcccc,stroke:#ff0000;

Apply :::failure class to any Node containing High Severity violations.

Use dotted lines -.-> for "Leaky Dependencies" (ACL violations).

```
graph TD
    classDef failure fill:#f96,stroke:#333,stroke-width:2px;
    
    Client -->|POST /pay| CheckoutService
    CheckoutService -->|Direct SQL| InventoryDB
    CheckoutService -->|Leaked Model| StripeAPI
    
    class CheckoutService failure
```


## Task 3: Generate resilience/remediation_plan.md
Create a prioritized Markdown list.

Section 1: Critical Fixes (Code blocking production reliability).

Section 2: Architectural Refactors (ACLs, Package moves).

Section 3: Next Steps (Specific CLI commands the user can run to apply fixes).