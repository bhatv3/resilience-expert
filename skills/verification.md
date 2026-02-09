# Skill: Verification (Evidence & Rigor)

## Purpose
Enforce rigor and prevent speculative or intuition-driven output in all resiliency analysis artifacts.

This skill defines the **non-negotiable standards** that apply to:
- discovery outputs (`resilience/discovery/discovery.json`, `arch-before.mmd`)
- lens findings (all lenses)
- backlog items and prioritization artifacts

Any output that violates this contract is considered invalid.

---

## Evidence Requirements
A claim is valid only if supported by **explicit evidence** from the repository.

Acceptable evidence includes at least one of:
- file path + line range
- configuration key + location in repo
- build manifest evidence (`pom.xml`, `build.gradle`, `go.mod`, `pyproject.toml`, etc.)
- explicit code reference (class/function usage) with location

Examples:
- “Dependency X has no timeout” must cite the client instantiation or call site and show absence of timeout configuration.
- “Circuit breaker exists” must cite the wrapper, annotation, or configuration that enables it.

Absence of evidence must be treated as **unknown**, not assumed behavior.

---

## Prohibited Behaviors
The following are explicitly disallowed:
- Inferring production topology (regions, AZs, deployment shape) unless explicitly present in code or configuration.
- Inferring SLOs, SLAs, or reliability guarantees without a cited source.
- Assuming default failure posture (fail-open / fail-closed) without evidence.
- Using vague or hedge language (“likely”, “probably”, “appears to”) for factual claims.

When evidence is insufficient:
- Use `UNKNOWN`
- Emit a validation question instead of a claim

---

## UNKNOWN Handling
When evidence is missing or ambiguous:
1) Mark the relevant field as `unknown`
2) Emit a validation question in:
    - `discovery.json` under `open_questions[]`, and/or
    - `role_assessment.md` under “Open Questions & Gaps”

Validation questions must:
- clearly state what is unknown
- point to relevant code or configuration areas
- explain why clarification matters (risk, impact, or decision dependency)

UNKNOWN is a **valid and expected outcome** of Phase 1 analysis.

---

## Confidence Scoring
Each finding must include a confidence level:

- **High** — direct, unambiguous evidence in code or configuration
- **Medium** — partial evidence; some ambiguity remains
- **Low** — weak or indirect evidence; must be framed as an investigation

Rules:
- Low-confidence findings must not produce P0 work items unless:
    - the failure mechanism itself is evidenced, and
    - the potential customer impact is severe
- Confidence must reflect evidence quality, not perceived importance.

---

## Recommendations vs Investigation
Not all findings require an immediate concrete fix.

If evidence is insufficient to recommend a specific change:
- The output should be an **investigation or validation task**, not a design proposal
- The task must clearly state:
    - what needs to be validated
    - what decision it will inform
    - why it matters for resiliency or regionalization

This preserves Phase 1’s analysis-first intent and avoids premature solutions.

---

## Customer Impact Guidance
Customer impact must describe **observable user-facing behavior**, such as:
- increased latency
- failed or delayed verifications
- retries, fallback, or degraded delivery
- partial loss of functionality

Avoid framing impact purely in internal or architectural terms unless explicitly tied to user-visible effects.

---

## Consistency Checks (Before Emitting Outputs)
Before writing final artifacts, ensure:
1) Every finding includes:
    - severity
    - confidence
    - evidence
    - customer impact
    - recommended change *or* investigation task
    - validation notes (if applicable)
2) Every backlog item maps 1:1 to a finding (no orphan tickets).
3) Diagrams reflect discovery facts only (no inferred topology or future state).
4) Phase 1 scope is respected (no multi-region, active-active, or deployment changes).

---

## Output Quality Bar
Outputs are acceptable only when:
- A reviewer can trace each finding directly to evidence
- Unknowns are explicitly captured and justified
- Recommendations are actionable or clearly investigative
- The backlog is coherent and directly importable into planning systems