# Agent Common Rules and Guidelines

This document contains the common rules and guidelines that apply to ALL specialized agents working on the AgentFlare platform.

## Core Principles

### KISS, DRY, YAGNI, and Zen Principles
- Keep implementations simple and straightforward
- Avoid code duplication
- Don't build features that aren't needed
- Follow the Zen of Python philosophy where applicable

### Clean Code and SOLID Principles
- Write self-documenting code with clear intent
- Apply Single Responsibility, Open-Closed, Liskov Substitution, Interface Segregation, and Dependency Inversion principles
- Maintain clean, readable, and maintainable code

### Twelve-Factor App Methodology
- Build cloud-native, scalable architectures
- Externalize configuration
- Treat backing services as attached resources
- Maximize portability between execution environments

## Quality Standards

### Test Coverage Requirements
- **MANDATORY**: 90% minimum test coverage for all code
- Write comprehensive unit and integration tests
- Use table-driven tests where applicable
- **ALWAYS EXECUTE ALL TESTS** after writing them
- **VERIFY ALL TESTS PASS** before marking any task complete
- **NEVER** submit code without running tests and confirming they pass

### Code Review Process
**IMPORTANT**: Agents cannot directly invoke other agents. You must report back to the orchestrating agent with your request.

When code review is needed:
1. Complete your implementation following all standards
2. Report back to the orchestrator: "Implementation complete. Requesting quality-assurance-engineer review."
3. The orchestrator will invoke quality-assurance-engineer
4. When you receive QA feedback, address ALL identified issues
5. Report back for another review cycle if needed
6. Continue until QA approval is confirmed
7. **ONLY** proceed to next steps after QA approval

## Project Boundaries

### CRITICAL: Cross-Project Isolation Rules

**IMMEDIATE STOP RULE**: If you identify an issue or need changes outside your designated directory:

1. **STOP IMMEDIATELY** - Do not proceed with modifications
2. **DO NOT** modify files outside your project directory
3. **REPORT** the exact issue and location to the main agent
4. **SPECIFY** which project engineer should handle it

### Boundary Enforcement

Your scope is **STRICTLY LIMITED** to your designated project directory:
- platform-engineer: `/platform` directory ONLY
- api-server-engineer: `/api-server` directory ONLY
- proxy-engineer: `/proxy` directory ONLY
- db-engineer: `/db` directory ONLY
- integration-engineer: `/integration` directory ONLY
- spec-engineer: `.spec.md` files in appropriate directories

### Cross-Project Communication

**REMEMBER**: You cannot directly invoke other agents. Report back to the orchestrator.

When you need support from other projects:
1. Document exact requirements (endpoints, functions, schemas)
2. Report to orchestrator: "Need [specific function] from [project]-engineer"
3. Wait for orchestrator to coordinate with the appropriate engineer
4. Resume work once the dependency is resolved
5. NEVER modify other projects' code yourself

### Response Template for Out-of-Scope Issues

```
"I've identified the issue/requirement: [description]. 
This requires changes in [project]/[file]:[line]. 
The [project]-engineer must handle this modification. 
I cannot modify files outside the [my-project]/ directory."
```

## Swimlane Todo Plan Workflow

When working from swimlane todo plans (`.claude/todos/*.todo.md`):

1. **Locate Your Column**: Find your role's column in the swimlane table
2. **Identify Current Row**: Start with topmost incomplete task
3. **Check Dependencies**: Look for incoming arrows (→) indicating prerequisites
4. **Execute Tasks**:
   - Mark as in-progress: Update checkbox to `[~]`
   - Complete implementation following all standards
   - Mark as complete: Update checkbox to `[x]`
5. **Handoff Work**: Notify receiving engineer when you see outgoing arrows (→)
6. **Parallel Execution**: Tasks in same row can be done simultaneously
7. **Update Plan**: Keep todo file updated with progress

## AgentFlare Platform Context

### Vision and Mission
- Building "The Cloudflare of AI Agents"
- Telepathy is the flagship observability feature
- Platform MUST be protocol-agnostic
- Target users: agent developers and server owners

### MVP Alignment
- All work must align with MVP scope in `contributing/scope.md`
- Core value: "Change one import line, instantly understand every agent decision"
- Support reasoning, performance, and cost tracking

### Protocol-Agnostic Design
- NEVER lock into single protocol implementations
- Use protocol registry patterns for extensibility
- Keep protocol-specific logic in `/protocols/[protocol-name]`
- Core components must remain protocol-agnostic

## Technical Standards

### Naming Conventions
- Follow language-specific conventions (Go, TypeScript, Python)
- Use AF prefix where required (e.g., AFButton, AFModal in React)
- Query constants: `insert<Domain><Entity>Query` pattern
- Function names: Insert*, Select*, Update*, Delete* for CRUD

### Performance Targets
- Proxy: 50K+ RPS, P95 < 10ms latency
- API Server: 10K+ RPS, P95 < 50ms
- Platform Dashboard: <100ms initial render
- WebSocket: <50ms latency

### OpenTelemetry Requirements
- **REQUIRED**: Comprehensive instrumentation throughout
- Every file must initialize tracer and meter
- Every significant operation must create spans
- Include metrics, traces, and logs

### Security Standards
- Never hardcode credentials or API keys
- Implement proper authentication and authorization
- Validate all inputs
- Prevent SQL injection and XSS
- Ensure multi-tenant isolation

## Documentation Requirements

### Code Documentation
- Document complex logic with clear comments
- Explain "why" not "what" in comments
- Include examples for complex APIs
- Document breaking changes clearly

### Architectural Documentation
- Document all architectural decisions
- Include rationale for technical choices
- Maintain up-to-date API contracts
- Create diagrams for complex flows

## Integration Testing Coordination

After QA approval, report to orchestrator for integration testing:
- Request: "QA approved. Ready for integration-engineer testing."
- The orchestrator will coordinate with integration-engineer for:
  - End-to-end testing with real services (no mocks)
  - Docker-based test environments
  - Playwright browser tests (platform features)
  - API integration tests
  - Performance validation under load
  - Multi-tenant isolation verification

## Prohibited Actions

**ABSOLUTELY PROHIBITED**:
- Modifying files outside your project directory
- "Helping out" by fixing issues in other projects
- Making "quick fixes" outside boundaries
- Using init() functions for metrics (use sync.OnceValue)
- Hardcoding values (use configuration)
- Skipping tests or quality reviews
- Submitting code without test execution
- Using external test frameworks when standard library suffices

## Decision Framework

When making technical decisions:
1. Align with AgentFlare vision and MVP scope
2. Follow KISS principle - simplicity first
3. Ensure protocol-agnostic design
4. Meet performance targets
5. Maintain 90% test coverage
6. Consider multi-tenant implications
7. Plan for scalability and future growth

## Communication Guidelines

### Clarity and Precision
- Provide specific, actionable feedback
- Include exact file locations and line numbers
- Suggest concrete improvements with examples
- Document important decisions and rationale

### Escalation Protocol
- Escalate architectural concerns early
- Use blocker templates when stuck
- Request clarification on requirements
- Coordinate cross-project dependencies properly

## Responding to Planner Agent Requests

When the planner agent requests task breakdown:
1. **Analyze Current State**: Review existing code and architecture
2. **Identify Required Changes**: Determine specific modifications needed
3. **Generate Task List** including:
   - Specific implementation tasks
   - Dependencies on other teams
   - What other teams need from you
   - Estimated effort per task
   - Potential risks or blockers
   - Testing requirements
   - Documentation needs

## Agent Coordination Protocol

**CRITICAL**: Agents cannot invoke other agents directly. All inter-agent coordination must go through the orchestrator.

### Requesting Another Agent's Help
When you need another agent's assistance:
1. **STOP** your current work at a logical checkpoint
2. **REPORT** to orchestrator with specific request
3. **WAIT** for orchestrator to coordinate
4. **RESUME** when dependency is resolved

### Example Coordination Messages
- "Implementation complete. Requesting quality-assurance-engineer review."
- "Need SelectUsersByWorkspace function from db-engineer."
- "Found issue in api-server/handlers.go:147. The api-server-engineer needs to fix the validation logic."
- "QA approved. Ready for integration-engineer testing."
- "Blocked: Need API endpoint /api/v1/metrics from api-server-engineer."

## Summary

These rules ensure consistent, high-quality development across all AgentFlare projects. Every agent must:
- Stay within project boundaries
- Report to orchestrator for cross-agent coordination
- Maintain 90% test coverage
- Execute and pass all tests
- Follow iterative QA review through orchestrator
- Coordinate for integration testing via orchestrator
- Align with platform vision
- Maintain protocol-agnostic design
- Meet performance targets
- Document decisions clearly