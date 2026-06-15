---
trigger: always_on
---

# XState Rules

- XSTATE-001: MUST model behavior with states. Never use boolean status flags in context.
- XSTATE-002: MUST avoid monolithic "God machines." Create separate actors for independent logic.
- XSTATE-003: MUST use hierarchical (parent) states to deduplicate shared transitions (e.g., global error/logout) and parallel states for concurrent concerns.
- XSTATE-004: MUST represent request lifecycles as discrete states (`idle`, `loading`, `success`, `error`).
- XSTATE-005: MUST use `dot.case` for events representing occurrences (e.g., `data.loaded`, `form.submit`). Never use command verbs (e.g., `setLoading`).
- XSTATE-006: MUST use wildcards (e.g., `user.*`) to handle grouped events.
- XSTATE-007: MUST store only essential identifiers and metadata (e.g., retry counters) in context. Do not store full domain objects or UI state.
- XSTATE-008: MUST never mutate context directly; always use `assign()`.
- XSTATE-009: MUST implement actions within the `setup()` function.
- XSTATE-010: MUST use `enqueueActions` for sequences requiring multiple mutations or conditional scheduling.
- XSTATE-011: MUST never use `if/else` inside actions. Always route logic via guarded transitions.
- XSTATE-012: MUST keep guards strictly pure. Move all side-effects (e.g., logging, I/O) to actions.
- XSTATE-013: MUST use `reenter: true` on transitions to reset timers for debounce patterns.
- XSTATE-014: MUST never use `sendParent()`. Pass necessary actor references (e.g., coordinator, siblings) explicitly via `input`.
- XSTATE-015: MUST use `systemId` to implement global broadcast, event bus, or gossip patterns.
- XSTATE-016: MUST use `invoke` when an actor is strictly tied to a specific state. Use `spawn` for dynamic actors requiring manual lifecycle control.