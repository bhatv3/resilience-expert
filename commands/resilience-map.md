---
description: Map the legacy system boundaries without analyzing them.
argument-hint: [scope]
---

# /resilience-map

## Objective
Execute **Phase 1 (Discovery)** only. Do not critique the code; just map it.

## Instructions
1. Invoke the **Legacy Module Discovery** skill (`skills/discovery.md`).
2. Identify:
   - **Virtual Modules** (Clusters of files touching the same DB tables).
   - **Entry Points** (REST, Kafka, SQS) and which Module they trigger.
3. **Artifact Output:**
   - Create `resilience/module_map.json`.
   - Create `resilience/system_map.mermaid`.
   
## Visual Output
Render the Mermaid chart immediately in the chat so the user can see the "Big Ball of Mud" structure.
