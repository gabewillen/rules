# HSM-BASE-001 MUST Apply Go And HSM Pattern Rules

See:
- [GO-CONC-001](go.rules.md#go-conc-001-must-own-goroutines)
- [PAT-HSM-001](patterns.rules.md#pat-hsm-001-must-explicit-hierarchical-state-modeling)

Go HSM code MUST comply with Go rules and hierarchical state machine pattern rules.

# HSM-INIT-001 MUST Define Initial Transitions

See:
- [PAT-HSM-001](patterns.rules.md#pat-hsm-001-must-explicit-hierarchical-state-modeling)

Every model and composite state that enters a nested substate MUST define an explicit initial transition.

Initial transitions MUST target nested states and MUST NOT use guards.

# HSM-STRUCT-001 MUST Keep State And Transition Declarations Valid

Transition fields such as source, target, trigger, guard, and effect MUST be declared in transition definitions.

State behavior such as entry, exit, activity, and defer MUST be declared in state definitions.

# HSM-FINAL-001 MUST Keep Final States Terminal

Final states MUST NOT define outgoing transitions, activities, entry actions, or exit actions.

# HSM-HISTORY-001 MUST Provide History Fallbacks

History pseudostates MUST live inside composite states.

History pseudostates MUST provide explicit fallback transitions for first-time re-entry.

# HSM-EVENT-001 MUST Use Explicit Triggers

See:
- [PAT-EVENT-001](patterns.rules.md#pat-event-001-must-typed-event-boundaries)

Every HSM transition MUST have an explicit trigger where the HSM API requires triggers.

String wildcards and implicit completion progression are forbidden unless modeled by the framework as explicit events.

# HSM-CHOICE-001 MUST Model Conditional Branching With Choices

See:
- [PAT-HSM-001](patterns.rules.md#pat-hsm-001-must-explicit-hierarchical-state-modeling)

Conditional behavioral branching MUST be modeled with choice states, guarded transitions, or explicit outcome events.

Choice states MUST have a deterministic fallback branch.

# HSM-GUARD-001 MUST Keep Guards Pure

See:
- [PAT-GUARD-001](patterns.rules.md#pat-guard-001-must-pure-guards)

HSM guards MUST be pure predicates.

Guards MUST NOT perform I/O, logging, allocation-heavy work, or state mutation.

# HSM-STATE-001 MUST Keep Durable State Machine Owned

See:
- [CORE-STATE-001](core.rules.md#core-state-001-must-single-source-of-truth)

Durable machine data MUST be owned by the machine instance, declared attributes, or explicit runtime data structures.

Caller context values MUST NOT store durable machine state.

# HSM-OBS-001 MUST Observe Through Snapshots

See:
- [PAT-SNAPSHOT-001](patterns.rules.md#pat-snapshot-001-must-snapshot-observation)

External code MUST observe HSM state through snapshots or subscriptions.

External code MUST NOT use observed transient states to manually drive internal progression.

# HSM-TIME-001 MUST Model Time Explicitly

See:
- [CORE-BOUND-001](core.rules.md#core-bound-001-must-explicit-platform-boundaries)

HSM behavior MUST NOT call sleeps, timers, wall clocks, or random sources directly.

Time MUST enter through modeled time events, injected clocks, or boundary adapters.

# HSM-ACTIVITY-001 MUST Bound Activities

See:
- [PAT-ASYNC-001](patterns.rules.md#pat-async-001-must-async-work-return-events)

Long-running HSM activities MUST have an owner, cancellation path, and explicit result events.

Short synchronous work SHOULD be modeled as actions rather than activities.

# HSM-DISPATCH-001 MUST Treat Async Dispatch As A Boundary

See:
- [CORE-WORK-001](core.rules.md#core-work-001-must-bounded-runtime-work)

Asynchronous dispatch, set, restart, stop, fanout, and directed dispatch operations MUST expose completion or failure to callers that depend on the result.

Callers MUST use cancellation-aware waits when waiting.

# HSM-CATCHALL-001 SHOULD Keep Catch-All Transitions Lowest Priority

Catch-all transitions SHOULD be lowest priority.

Catch-all transitions MUST NOT accidentally consume internal lifecycle events.
