# Domain Boundaries

## Architecture & Package Ownership
1. **Keep Generic Packages Pure:** Core packages (`platform`, `block`, `chat`, `gateway`, `surface`) must not contain connector-specific logic, credential fields, or domain routing. Delegate these contracts to specific agent/domain packages.
2. **No Hidden Dispatch Paths:** If a package does not own a workflow, it cannot contain private HSM events, tool-result mappings, action IDs, or fallback branches for that workflow.
3. **No Unauthorized Packages:** Never create new packages, modules, or top-level boundaries without explicit user approval in the active conversation.

## UI & Presentation
1. **Strict Display Name Resolution:** Never infer persona or display names (e.g., "Conner", "Heidi", "Plato") from refs, tool names, action IDs, or string substrings. 
2. **Rely on Explicit Metadata:** Always use explicit runtime participant display-name metadata. Fallback to generic agent or role labels if unavailable.

## Testing & Verification
1. **Deep Boundary Verification:** Standard boundary scripts are insufficient. Plans and verification steps must explicitly include scanning imports, strings, proxy paths, comments, tests, documentation, selectors, and fixture names for domain leakage.
