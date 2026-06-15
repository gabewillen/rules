---
alwaysApply: true
---

# HSM Rules

- HSM-001: MUST define `hsm.Initial(...)` for every model and composite state that enters a nested substate. Initial transitions MUST target a nested state, have exactly one outgoing transition, and MUST NOT use guards.
- HSM-002: MUST NOT put entry or exit actions on the top-level machine. Top-level transitions MUST specify both `Source` and `Target`, or neither.
- HSM-003: MUST place history pseudostates only inside composite states. MUST always provide an explicit fallback transition for first-time re-entry.
- HSM-004: MUST NOT attach transitions, activities, entry actions, or exit actions to a final state.
- HSM-005: MUST decompose complex state machines into smaller behavioral units to avoid state explosion.
- HSM-006: MUST declare `Source`, `Target`, `On`, `OnSet`, `OnCall`, `After`, `Every`, `When`, `Guard`, and `Effect` strictly inside `hsm.Transition(...)`. State paths MUST be valid relative or absolute paths.
- HSM-007: MUST declare `Entry`, `Exit`, `Activity`, and `Defer` strictly inside `hsm.State(...)`.
- HSM-008: MUST NOT use empty strings for attribute, operation, `OnSet`, or `OnCall` names. MUST NOT define duplicate attributes or operations in the same model.
- HSM-009: MUST require an explicit trigger for every transition. Implicit completion transitions are invalid. Use explicit events, not string wildcards.
- HSM-010: MUST use explicit events with `Kind: hsm.CompletionEventKind` to trigger prioritized follow-on progression from actions or guards. Use `Kind: hsm.ErrorEventKind` for machine-owned errors.
- HSM-011: MUST model conditional branching with `hsm.Choice(...)`, not via `if/switch` logic inside actions. External calls may dispatch typed outcome events, but resulting state branches MUST be explicitly modeled in the HSM. Choices MUST have outgoing transitions, ending with an unguarded default branch.
- HSM-012: MUST use `hsm.On(hsm.AnyEvent)` only as a lowest-priority catch-all. Guard it against internal lifecycle events unless intended.
- HSM-013: MUST order multiple transitions for the same event from highest-priority guarded branch to lowest-priority fallback. The first passing transition wins.
- HSM-014: MUST use internal transitions (`Source` with no `Target`) to run an `Effect` without changing state. MUST NOT use self-reentry transitions (`Target` equals `Source`) unless deliberate entry/exit/activity restart semantics are required.
- HSM-015: MUST use `hsm.Defer(...)` to postpone an event until the machine leaves its current state.
- HSM-016: MUST NOT use mutexes. The Run-to-Completion (RTC) model strictly serializes state access. Prefer events and attributes over shared memory synchronization.
- HSM-017: MUST keep private bookkeeping as machine-owned struct fields mutated only inside behavior. Move external data into the machine via events/attributes. MUST NOT store machine attributes or durable state in `context.Context`.
- HSM-018: MUST NOT expose internal state via public getters. External code MUST use `hsm.TakeSnapshot(...)` at service boundaries to observe state.
- HSM-019: MUST NOT inspect `TakeSnapshot(...).State` or `State()` from external code to manually drive business progression. Transient state progression belongs strictly inside the HSM model.
- HSM-020: MUST use `ID()` for stable routing and `Name()` for the model name. MUST NOT use `Name()` as a unique instance ID. Use `QualifiedName()` for the full model path.
- HSM-021: MUST expose reactive data using `hsm.Attribute(...)`. MUST only modify attributes via `Set()` to trigger `hsm.OnSet(...)` transitions. Do not expect setting an identical value to trigger a change event.
- HSM-022: MUST define `hsm.Operation(...)` before handling it with `hsm.OnCall(...)`. Trigger operations strictly via `hsm.Call(...)` instead of manually dispatching synthetic call events.
- HSM-023: MUST NOT use Go timers (`time.Sleep`, `time.After`, `time.NewTicker`) inside HSM behavior. Model time explicitly using `hsm.After(...)`, `hsm.At(...)`, or `hsm.Every(...)` on transitions from real states.
- HSM-024: MUST reserve `hsm.Activity(...)` for long-running or continuously waiting work, not short synchronous work. Activities MUST promptly respect `ctx.Done()` to guarantee clean termination.
- HSM-025: MUST NOT call `time.Now()`, random sources, filesystem, network, or environment APIs directly from behavior functions. Inject dependencies (e.g., `ClockFn`) or pass external results via event payloads.
- HSM-026: MUST prefer pass-by-value event payloads; use pointers only for large structs. If atomics are required, use value-semantic wrappers (e.g., `internal/generic` `Atomic`), not stdlib typed atomics.
- HSM-027: MUST use payloads with a buffered size-1 channel for req/resp. Follow strict ordering: dispatch the event, wait for dispatch completion, then read the response. Both sender and receiver MUST use a cancellation-aware `select`.
- HSM-028: MUST treat `hsm.Dispatch`, `hsm.Set`, `hsm.Restart`, `hsm.Stop`, `hsm.DispatchAll`, and `hsm.DispatchTo` as async boundaries. Always wait for their completion signal using a cancellation-aware `select`. Never use lifecycle hooks like `hsm.AfterProcess` for synchronization.
- HSM-029: MUST normalize nil dispatch contexts to `context.Background()`. For fire-and-forget events, never use a transient caller context; use `hsm.Context()` if bound to the machine's lifetime, or `context.Background()` if it must outlive the machine.
- HSM-030: MUST NOT block on a self-dispatched follow-up event from inside machine behavior if the follow-up represents asynchronous protocol progression.
- HSM-031: MUST use `hsm.Started(...)` with explicit `hsm.Config` identity for behavior-owning runtime instances, including `Data` if startup input is needed.
