# AgentFlare Core Rules

## 1. Architecture & Code Quality
1. **Design Principles**: Apply KISS, DRY, YAGNI, SOLID, and 12-Factor App standards. Code must be clean and self-documenting. Comment the "why", not the "what". Document architectural decisions and maintain up-to-date API contracts.
2. **Protocol-Agnostic Core**: Isolate protocol-specific logic within `/protocols/[name]`. Use registry patterns for extensibility.
3. **Telemetry**: Instrument comprehensively with OpenTelemetry (metrics, traces, logs). Initialize tracers/meters per file and span all significant operations. Use `sync.OnceValue` instead of `init()` for metrics.
4. **Security**: Externalize all configuration. Never hardcode secrets or config values. Enforce strict multi-tenant isolation, validate all inputs, and prevent SQLi/XSS.
5. **Performance Requirements**:
   - Proxy: >50K RPS, P95 <10ms
   - API Server: >10K RPS, P95 <50ms
   - WebSocket: <50ms latency
   - Dashboard: <100ms initial render
6. **Naming Standards**: Follow language-native conventions. Prefix React components with `AF` (e.g., `AFModal`). Format database queries as `[Action]<Domain><Entity>Query`.

## 2. Project Boundaries
1. **Strict Directory Isolation**: You may **ONLY** modify files within your assigned project directory (e.g., `/platform`, `/api-server`, `/proxy`, `/db`, `/integration`, `.spec.md` files).
2. **Out-of-Scope Protocol**: If a modification is required outside your directory, **STOP IMMEDIATELY**. Do not write "quick fixes" or "help out" other teams. Report the exact file, line number, requirement, and responsible engineer to the orchestrator.
3. **Strategic Alignment**: All work must support the MVP scope (`contributing/scope.md`) and the core vision: *"Change one import line, instantly understand every agent decision."*

## 3. Workflow & Agent Coordination
1. **Centralized Orchestration**: Agents cannot invoke or directly assist each other. All inter-agent coordination, dependency requests, and handoffs MUST route through the orchestrating agent.
2. **Dependency & Escalation**: If blocked by missing cross-team functionality, escalate early using blocker templates. Document the specific requirement, report it to the orchestrator, and wait for resolution.
3. **Swimlane Execution (`.claude/todos/*.todo.md`)**:
   - Work top-down in your designated role column.
   - Respect incoming dependency arrows (`→`).
   - Mark tasks `[~]` (in-progress) and `[x]` (complete).
   - Upon completion, notify the receiving engineer via the orchestrator for outgoing dependencies (`→`).
4. **Task Breakdowns**: When a planner requests task details, provide required code changes, cross-team dependencies, effort estimates, risks, and testing/documentation requirements.

## 4. Testing & Quality Assurance
1. **Coverage & Execution**: Maintain a mandatory 90% test coverage minimum. Use table-driven tests and standard libraries over external frameworks. **ALWAYS** execute all tests and ensure they pass before submitting code.
2. **Code Review Cycle**: After implementation, request a `quality-assurance-engineer` review via the orchestrator. Address all feedback iteratively until you receive explicit QA approval.
3. **Integration Phase**: Following QA approval, request `integration-engineer` testing via the orchestrator. This phase requires real services (no mocks), Docker environments, Playwright browser tests, multi-tenant verification, and load testing.