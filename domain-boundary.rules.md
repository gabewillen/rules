# Domain Boundary Rules

- UI code must not infer persona or display names such as Conner, Heidi, Oneil, or Plato from participant refs, role refs, tool names, action IDs, or string substrings. User-facing agent names must come from explicit runtime participant display-name metadata; otherwise render stable role labels or generic agent labels.
- Generic platform, block, chat, gateway, and surface packages must not own connector-specific setup logic, credential field names, or domain-specific routing events when an agent/domain package can own that contract.
- Hidden dispatch paths count as boundary exposure. If a package must not own a domain workflow, it must not keep private HSM events, tool-result mappings, action IDs, or fallback branches for that workflow.
- Do not create a new package, module, submodule, service package, adapter package, frontend runtime package, generated package, or top-level package-like boundary without explicit user approval in the active conversation.
- A clean boundary script is not sufficient. Plans and verification must classify manual import/string/proxy-path scans, including comments, tests, docs, selectors, fixture names, deterministic inference scripts, and compatibility paths.
