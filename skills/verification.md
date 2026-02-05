# Skill: Verification (Evidence & Rigor)

## Purpose
Enforce rigor and prevent speculative output in all resiliency analysis artifacts.

This skill applies to:
- discovery outputs (`resilience/discovery.json`, `arch-before.mmd`)
- analysis findings (all lenses)
- backlog items and prioritization

---

## Evidence Requirements
A claim is valid only if supported by at least one of:
- file path + line range
- configuration key + location in repo
- build manifest evidence (pom.xml, gradle, go.mod, package.json)
- explicit code reference (class/function usage) with location

Examples:
- "Dependency X has no timeout" must cite call site and show absence of timeout configuration.
- "Circuit breaker exists" must cite the configuration or wrapper usage.

---

## Prohibited Behaviors
- Do not infer production topology (regions, AZs, deployment shape) unless explicitly present in repo/config.
- Do not infer SLOs/SLA targets without a cited source.
- Do not assume default failure posture (fail-open/closed) unless evidence exists.
- Do not use vague language ("likely", "probably") for factual claims.
    - Use UNKNOWN + a validation question instead.

---

## UNKNOWN Handling
When evidence is missing or ambiguous:
1) Mark the field as `unknown`
2) Emit a validation question in:
    - `discovery.json` under `open_questions[]`, and/or
    - `role_assessment.md` under "Open Questions & Gaps"

Validation questions must:
- state what is unknown
- point to relevant code/config areas
- explain why it matters (risk/impact)

---

## Confidence Scoring
Each finding must include confidence:
- High: direct evidence in code/config
- Medium: partial evidence; some ambiguity remains
- Low: weak evidence; must be framed as a question or investigation candidate

Low confidence findings should not produce P0 work items unless the risk is clearly severe and evidence supports the risk mechanism.

---

## Consistency Checks (Before Emitting Outputs)
Before writing final artifacts:
1) Ensure every finding includes:
    - severity, confidence, evidence, customer impact, recommended change, validation
2) Ensure backlog items map to findings (no orphan tickets).
3) Ensure diagrams reflect discovery facts only (no proposed topology changes).
4) Ensure Phase 1 scope is respected (no multi-region / active-active proposals).

---

## Output Quality Bar
Outputs are acceptable only if:
- A reviewer can trace each finding to evidence
- Unknowns are explicitly captured
- Recommendations are actionable and scoped to Phase 1
- The backlog is coherent and importable into planning