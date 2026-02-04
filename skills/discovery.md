# Skill: Legacy Module Discovery

## Objective
Map logical boundaries in non-modular codebases ("Big Ball of Mud").

## Heuristics
1.  **Gravity Clustering:**
    * Identify "Core Entities" (e.g., Database Tables, Protobuf definitions).
    * Group code files that *write* to these entities into a **Virtual Module**.
    * *Example:* If 5 files all write to the `orders` table, label them `Module::Orders`.

2.  **Shared Persistence Detection:**
    * Identify if multiple Virtual Modules write to the *same* database table.
    * Flag this as **"Shared Database Integration"** (High Risk).

3.  **The "God Class" Detector:**
    * Identify files with >50 incoming dependencies from different directories.
    * Label these as **"Coupling Hotspots."**