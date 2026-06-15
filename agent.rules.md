# AgentFlare Core Rules

- AGENT-001: MUST apply KISS, DRY, YAGNI, SOLID, and 12-Factor App standards. Code MUST be clean and self-documenting. Comment the "why", not the "what". Document architectural decisions and maintain up-to-date API contracts.
- AGENT-002: MUST isolate protocol-specific logic within `/protocols/[name]`. Use registry patterns for extensibility.
- AGENT-003: MUST instrument comprehensively with OpenTelemetry (metrics, traces, logs). Initialize tracers/meters per file and span all significant operations. Use `sync.OnceValue` instead of `init()` for metrics.
- AGENT-004: MUST externalize all configuration. Never hardcode secrets or config values. Enforce strict multi-tenant isolation, validate all inputs, and prevent SQLi/XSS.
- AGENT-005: MUST meet these performance targets: Proxy >50K RPS with P95 <10ms; API Server >10K RPS with P95 <50ms; WebSocket <50ms latency; Dashboard <100ms initial render.
- AGENT-006: MUST follow language-native naming conventions. Prefix React components with `AF` (e.g., `AFModal`). Format database queries as `[Action]<Domain><Entity>Query`.
- AGENT-007: MUST ONLY modify files within the assigned project directory (e.g., `/platform`, `/api-server`, `/proxy`, `/db`, `/integration`, `.spec.md` files). 
- AGENT-008: MUST immediately STOP if a modification is required outside the assigned directory. Do not write "quick fixes" or "help out" other teams. Report the exact file, line number, requirement, and responsible engineer to the orchestrator.
- AGENT-009: MUST ensure all work supports the MVP scope (`contributing/scope.md`) and the core vision: "Change one import line, instantly understand every agent decision."
- AGENT-010: MUST NOT allow agents to invoke or directly assist each other. Route all inter-agent coordination, dependency requests, and handoffs through the orchestrating agent.
- AGENT-011: MUST escalate early using blocker templates when blocked by missing cross-team functionality. Document the specific requirement, report it to the orchestrator, and wait for resolution.
- AGENT-012: MUST execute work top-down in the designated role column of `.claude/todos/*.todo.md` files. Respect incoming dependency arrows (`→`). Mark tasks `[~]` (in-progress) and `[x]` (complete). Notify the receiving engineer via the orchestrator upon completion for outgoing dependencies.
- AGENT-013: MUST provide required code changes, cross-team dependencies, effort estimates, risks, and testing/documentation requirements when a planner requests task details.
- AGENT-014: MUST maintain a mandatory 90% test coverage minimum. Use table-driven tests and standard libraries over external frameworks. ALWAYS execute all tests and ensure they pass before submitting code.
- AGENT-015: MUST request a `quality-assurance-engineer` review via the orchestrator after implementation. Address all feedback iteratively until explicit QA approval is received.
- AGENT-016: MUST request `integration-engineer` testing via the orchestrator after QA approval. This phase requires real services (no mocks), Docker environments, Playwright browser tests, multi-tenant verification, and load testing.