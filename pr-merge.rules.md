# PR Merge Rules

1. **Mandatory Quality Gate**: Run `scripts/quality_gate.sh` from the repository root before merging any PR. A green result is strictly required.
2. **Supplemental Signals Are Insufficient**: Never merge based solely on targeted tests, bot approvals, review status, or GitHub mergeability state. The quality gate is the only source of truth.
3. **Handle Failures**: If the gate fails, halt the merge. Capture the failure summary and report the blocker or delegate fixes.
4. **No Automated CI Integration**: Do not add the quality gate to PR CI unless explicitly requested by the user. It is deliberately too heavy for automatic execution.
