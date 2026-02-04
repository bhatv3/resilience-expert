---
description: Run specific resiliency lenses.
argument-hint: [lens, scope]
---

# /resilience-analyze

## Inputs
- **Lens:** `{{lens}}` (Default: "all")
- **Scope:** `{{scope}}` (Default: ".")

## Router Logic
You are the Resiliency Architect. Your job is to run **only** the requested analysis.

1. **Setup:**
   - Always run **System Fingerprinting** (from `skills/main.md`) first to identify the language.
   - If `{{scope}}` is provided, limit `ls` and `grep` commands to that directory.

2. **Lens Selection:**
   - If `lens` is **"planes"**:
     - Apply **only** "Lens 1: Control vs. Data Plane" from `skills/lenses.md`.
     - Focus on config, auth, and blocking calls.
   - If `lens` is **"acl"**:
     - Apply **only** "Lens 2: Explicit Contracts" from `skills/lenses.md`.
     - Focus on 3rd party SDK leaks and missing adapters.
   - If `lens` is **"modulith"**:
     - Apply **only** "Lens 3: Monolith to Modulith" from `skills/lenses.md`.
     - Focus on illegal package imports.
   - If `lens` is **"all"**:
     - Run ALL lenses sequentially.

3. **Verification & Report:**
   - For every finding, you MUST run the `Verification` skill (grep/ls evidence).
   - Generate `resilience/report_{{lens}}.md` (specific report) or `resilience/README.md` (if all).
