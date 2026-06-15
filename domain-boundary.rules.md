# Domain Boundaries

- DOMAIN-001: MUST keep generic core packages (`platform`, `block`, `chat`, `gateway`, `surface`) free of connector-specific logic, credential fields, or domain routing. Delegate such contracts to specific agent or domain packages.
- DOMAIN-002: MUST ensure that if a package does not own a workflow, it does not contain private HSM events, tool-result mappings, action IDs, or fallback branches for that workflow.
- DOMAIN-003: MUST NOT create new packages, modules, or top-level boundaries without explicit user approval in the active conversation.
- DOMAIN-004: MUST never infer persona or display names (e.g., "Conner", "Heidi", "Plato") from refs, tool names, action IDs, or string substrings.
- DOMAIN-005: MUST rely on explicit runtime participant display-name metadata. Fall back to generic agent or role labels only if no explicit metadata is available.
- DOMAIN-006: MUST perform deep boundary verification in plans and reviews. Standard boundary scripts are insufficient; explicitly scan imports, strings, proxy paths, comments, tests, documentation, selectors, and fixture names for domain leakage.
