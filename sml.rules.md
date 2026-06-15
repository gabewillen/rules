# SML Architecture & Coding Rules

These rules establish a high-performance, real-time friendly architecture using SML (boost.ext/SML or stateforward.SML). The design behaves like a pure actor model while remaining fully synchronous, run-to-completion (RTC), and queue-free.

- SML-001: MUST treat dispatch as strictly synchronous run-to-completion (RTC). A top-level `process_event` MUST execute all internal and anonymous transitions to quiescence before returning.
- SML-002: MUST NOT use `process_queue`, `defer_queue`, mailboxes, or async "post-for-later" mechanisms.
- SML-003: MUST guarantee determinism: identical initial state + identical event sequence + identical payloads produces identical execution paths and state changes.
- SML-004: MUST enforce the single-writer rule: exactly one thread may execute inside an actor's `process_event` at any given time.
- SML-005: MUST forbid heap allocation during dispatch.
- SML-006: MUST ensure every dispatch has a provable upper bound on execution time and total transitions.
- SML-007: MUST use small, trivially copyable event types containing only immutable payloads. Do not use owning pointers or dynamic containers (`std::string`, `std::vector`) in events.
- SML-008: MUST ensure event payloads remain valid for the entire top-level `process_event` lifecycle (including nested sync dispatch).
- SML-009: MUST use typed completions (`sml::completion<TEvent>`) to propagate event data between internal phases. Internal-only events MAY carry mutable references strictly for same-RTC handoff.
- SML-010: MUST prefer compile-time typed dispatch. When runtime IDs are required, use static jump tables (`make_dispatch_table`) and validate ID bounds prior to indexing.
- SML-011: MUST treat states as control-flow labels. Store data in explicit, stable-address context objects injected via the state machine constructor.
- SML-012: MUST use context for persistent, machine-owned state. Do not mirror ephemeral event payloads into context just to pass data between phases.
- SML-013: MUST model error progression with explicit error states and events. Do not use context status fields as hidden control state.
- SML-014: MUST use `visit_current_states` or `is(...)` for external observation. Do not require locks in the steady state.
- SML-015: MUST ensure guards are pure predicates returning `bool`. Context mutation is restricted to actions.
- SML-016: MUST NOT allow actions or helpers to contain runtime routing logic (`if`, `switch`, loops deciding execution paths, success/error/mode decisions). Moving logic to a helper does not make it compliant. All branching MUST be modeled as guarded transitions or explicit choice states.
- SML-017: MAY use compile-time conditionals (`#if`, `if constexpr`) and bounded, monotonic data-plane iteration loops in actions/helpers.
- SML-018: MUST keep actions short, non-blocking (no I/O, no mutex waits), and ideally `noexcept`. Split long-running tasks using "waiting" states.
- SML-019: MUST catch unhandled external events using `sml::unexpected_event<...>`. Do not use wildcard `sml::_` guards that consume internal events.
- SML-020: MUST NOT allow an actor to call its own `process_event` from within a guard or action. Model internal microflows with typed completions or acyclic anonymous transitions.
- SML-021: MAY allow actor A to directly dispatch to actor B synchronously, provided the call graph is acyclic and orchestrator-ordered.
- SML-022: MAY use synchronous callbacks (`emel::callback` style) for immediate, same-RTC notification. MUST NOT store callbacks in context for later execution.
- SML-023: MUST avoid ad-hoc shared base classes for machine wrappers. Submachines MUST obey RTC/no-queue rules. Orthogonal regions MUST NOT have conflicting side effects.
- SML-024: MUST drive actors by an external orchestrator. Inject time via event payloads (e.g., `tick{now_ns}`). Store deadlines in context and evaluate via guards.
- SML-025: MAY allow coroutines (`co_sm`) by explicit opt-in only. Frame capacity MUST be pre-allocated. `co_await` MUST map exactly to SML phase boundaries (states/transitions) and MUST NOT hide inside guards, hot numeric loops, or routing helpers.
- SML-026: SHOULD use `sml::utility::sm_pool` when a single router machine must drive a large pool of logical actors/items.
- SML-027: MUST ensure `sm_pool` storage is default-constructible or constructible from a pool size.
- SML-028: MUST route by ID using `sml::utility::indexed_event`. Use contiguous batch APIs (`process_event_batch`). Validate IDs before indexing and avoid iterator overhead. Do not allocate during batch processing.
- SML-029: MUST use `PascalCase` for public C++ types and `lower_snake_case` for internal types, states, and events. Do not use macros in models.
- SML-030: MUST use destination-first syntax for transitions (`sml::state<dst> <= src + event [guard] / action`). Keep the destination and `<=` on the same line.
- SML-031: MUST use leading-comma style for multi-row tables. Wrap complex tables in `// clang-format off` and `// clang-format on` to prevent auto-formatting degradation. Break large tables into visual sections.
- SML-032: MUST explicitly assert determinism, bounded anonymous transitions, zero allocations (instrument global `new/delete`), and strict time budget compliance in tests.
- SML-033: MUST force states via `sml::testing::sm` in unit tests. Use a single-threaded orchestrator in integration tests to replay exact production dispatch orders.
