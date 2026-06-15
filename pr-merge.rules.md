# PR Merge Rules

- PR-MERGE-001: Before merging any PR, run `scripts/quality_gate.sh` from the repository root and require a green result.
- PR-MERGE-002: Do not merge a PR based on targeted tests, Bugbot, review status, or mergeable state alone. Those signals can inform review, but they never replace the root quality gate.
- PR-MERGE-003: If `scripts/quality_gate.sh` fails, do not merge. Capture the failure summary, then delegate fixes or report the blocker with the failing gate evidence.
- PR-MERGE-004: Do not add `scripts/quality_gate.sh` as a PR CI requirement unless the user explicitly approves it; the gate is intentionally too heavy for automatic CI.
