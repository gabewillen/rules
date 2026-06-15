# PR Merge Rules

- PR-001: MUST run `scripts/quality_gate.sh` from the repository root before merging any PR. A green result is strictly required.
- PR-002: MUST NOT merge based solely on targeted tests, bot approvals, review status, or GitHub mergeability state. The quality gate is the only source of truth.
- PR-003: MUST halt the merge if the quality gate fails, capture the failure summary, and report the blocker or delegate fixes.
- PR-004: MUST NOT add the quality gate to PR CI unless explicitly requested by the user. It is deliberately too heavy for automatic execution.
