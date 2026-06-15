---
alwaysApply: true
---

# HSM Rules

## 1. Machine Structure & Topology

1. **Initial States:** Define `hsm.Initial(...)` for every model and composite state that enters a nested substate. Initial transitions must target a nested state, have exactly one outgoing transition, and never use guards.
2. **Top-level Restrictions:** Never put entry or exit actions on the top-level machine. Top-level transitions must specify both `Source` and `Target`, or neither.
3. **History States:** Place history pseudostates only inside composite states. Always provide an explicit fallback transition for first-time re-entry.
4. **Final States:** Never attach transitions, activities, entry actions, or exit actions to a final state.
5. **Decomposition:** Decompose complex state machines into smaller behavioral units to avoid state explosion.

## 2. Declarations & Naming

1. **Transition Scope:** Declare `Source`, `Target`, `On`, `OnSet`, `OnCall`, `After`, `Every`, `When`, `Guard`, and `Effect` strictly inside `hsm.Transition(...)`. State paths must be valid relative or absolute paths.
2. **State Scope:** Declare `Entry`, `Exit`, `Activity`, and `Defer` strictly inside `hsm.State(...)`.
3. **Naming:** Never use empty strings for attribute, operation, `OnSet`, or `OnCall` names. Do not define duplicate attributes or operations in the same model.

## 3. Transitions & Branches

1. **Explicit Triggers:** Every transition requires an explicit trigger. Implicit completion transitions are invalid. Use explicit events, not string wildcards.
2. **Completion & Errors:** Use explicit events with `Kind: hsm.CompletionEventKind` to trigger prioritized follow-on progression from actions or guards. Use `Kind: hsm.ErrorEventKind` for machine-owned errors.
3. **Choices:** Model conditional branching with `hsm.Choice(...)`, not via `if/switch` logic inside actions. External calls can dispatch typed outcome events, but resulting state branches must be explicitly modeled in the HSM. Choices must have outgoing transitions, ending with an unguarded default branch.
4. **AnyEvent Fallback:** Use `hsm.On(hsm.AnyEvent)` only as a lowest-priority catch-all. Guard it against internal lifecycle events unless intended.
5. **Priority:** Order multiple transitions for the same event from highest-priority guarded branch to lowest-priority fallback. The first passing transition wins.
6. **Internal vs. Re-entry:** Use internal transitions (`Source` with no `Target`) to run an `Effect` without changing state. Never use self-reentry transitions (`Target` equals `Source`) unless deliberate entry/exit/activity restart semantics are required.
7. **Deferred Events:** Use `hsm.Defer(...)` to postpone an event until the machine leaves its current state.

## 4. State Encapsulation & Concurrency

1. **RTC Model:** Never use mutexes. The Run-to-Completion (RTC) model strictly serializes state access. Prefer events and attributes over shared memory synchronization.
2. **Data Isolation:** Keep private bookkeeping as machine-owned struct fields mutated only inside behavior. Move external data into the machine via events/attributes. Never store machine attributes or durable state in `context.Context`.
3. **No External Introspection:** Never expose internal state via public getters. External code must use `hsm.TakeSnapshot(...)` at service boundaries to observe state.
4. **Progression Ownership:** Never inspect `TakeSnapshot(...).State` or `State()` from external code to manually drive business progression. Transient state progression belongs strictly inside the HSM model.
5. **Identity:** Use `ID()` for stable routing and `Name()` for the model name. Never use `Name()` as a unique instance ID. Use `QualifiedName()` for the full model path.

## 5. Attributes & Operations

1. **Attributes:** Expose reactive data using `hsm.Attribute(...)`. Only modify attributes via `Set()` to trigger `hsm.OnSet(...)` transitions. Do not expect setting an identical value to trigger a change event.
2. **Operations:** Define `hsm.Operation(...)` before handling it with `hsm.OnCall(...)`. Trigger operations strictly via `hsm.Call(...)` instead of manually dispatching synthetic call events.

## 6. Time, Activities & Pure Functions

1. **Explicit Time Models:** Never use Go timers (`time.Sleep`, `time.After`, `time.NewTicker`) inside HSM behavior. Model time explicitly using `hsm.After(...)`, `hsm.At(...)`, or `hsm.Every(...)` on transitions from real states.
2. **Activities:** Reserve `hsm.Activity(...)` for long-running or continuously waiting work, not short synchronous work. Activities must promptly respect `ctx.Done()` to guarantee clean termination.
3. **Pure Behavior:** Never call `time.Now()`, random sources, filesystem, network, or environment APIs directly from behavior functions. Inject dependencies (e.g., `ClockFn`) or pass external results via event payloads.

## 7. Dispatch & Communication

1. **Payload Semantics:** Prefer pass-by-value event payloads; use pointers only for large structs. If atomics are required, use value-semantic wrappers (e.g., `internal/generic` `Atomic`), not stdlib typed atomics.
2. **Request/Response:** Use payloads with a buffered size-1 channel for req/resp. Follow strict ordering: dispatch the event, wait for dispatch completion, then read the response. Both sender and receiver must use a cancellation-aware `select`.
3. **Async Boundaries:** Treat `hsm.Dispatch`, `hsm.Set`, `hsm.Restart`, `hsm.Stop`, `hsm.DispatchAll`, and `hsm.DispatchTo` as async boundaries. Always wait for their completion signal using a cancellation-aware `select`. Never use lifecycle hooks like `hsm.AfterProcess` for synchronization.
4. **Context Usage:** Normalize nil dispatch contexts to `context.Background()`. For fire-and-forget events, never use a transient caller context; use `hsm.Context()` if bound to the machine's lifetime, or `context.Background()` if it must outlive the machine.
5. **Self-Dispatching:** Never block on a self-dispatched follow-up event from inside machine behavior if the follow-up represents asynchronous protocol progression.
6. **Instance Startup:** Use `hsm.Started(...)` with explicit `hsm.Config` identity for behavior-owning runtime instances, including `Data` if startup input is needed.
