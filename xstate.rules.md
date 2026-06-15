---
trigger: always_on
---

# XState Rules

## 1. Architecture & Modeling
1. **Use Explicit States:** Model behavior with states. Never use boolean status flags in context.
2. **Decompose by Concern:** Avoid monolithic "God machines." Create separate actors for independent logic.
3. **Hierarchy & Concurrency:** Use hierarchical (parent) states to deduplicate shared transitions (e.g., global error/logout) and parallel states for concurrent concerns.
4. **Model Async Explicitly:** Represent request lifecycles as discrete states (`idle`, `loading`, `success`, `error`).

## 2. Events
1. **Semantic Naming:** Use `dot.case` for events representing occurrences (e.g., `data.loaded`, `form.submit`). Never use command verbs (e.g., `setLoading`).
2. **Event Groups:** Use wildcards (e.g., `user.*`) to handle grouped events.

## 3. Context & Actions
1. **Keep Context Minimal:** Store only essential identifiers and metadata (e.g., retry counters). Do not store full domain objects or UI state.
2. **Immutable Context:** Never mutate context directly; always use `assign()`.
3. **Action Placement:** Implement actions within the `setup()` function.
4. **Complex Mutations:** Use `enqueueActions` for sequences requiring multiple mutations or conditional scheduling.

## 4. Guards & Transitions
1. **No Logic in Actions:** Never use `if/else` inside actions. Always route logic via guarded transitions.
2. **Pure Guards:** Guards must be strictly pure. Move all side-effects (e.g., logging, I/O) to actions.
3. **Debouncing:** Use `reenter: true` on transitions to reset timers for debounce patterns.

## 5. Actors & Communication
1. **Pass Actor Refs via Input:** Never use `sendParent()`. Pass necessary actor references (e.g., coordinator, siblings) explicitly via `input`.
2. **Global Pub-Sub:** Use `systemId` to implement global broadcast, event bus, or gossip patterns.
3. **Manage Lifecycles:** Use `invoke` when an actor is strictly tied to a specific state. Use `spawn` for dynamic actors requiring manual lifecycle control.