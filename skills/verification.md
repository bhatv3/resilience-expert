# Skill: Evidence-Based Verification

## Objective
Ensure every reported finding is backed by hard evidence found in the file system.

## Protocol
Before adding ANY finding to the report, you MUST:
1.  **Verify Existence:** Run `ls` to confirm the file path is valid.
2.  **Verify Content:** Run `grep -nC 2 "<pattern>" <file>` to confirm the code actually exists at that line.
3.  **Quote It:** Extract the exact snippet.

## Guardrails
- If you suspect a missing timeout, verify the specific HTTP client configuration file. Do not assume defaults.
- If you cannot find the file/line number, DISCARD the finding. Do not guess.