# SML Architecture & Coding Rules

These rules establish a high-performance, real-time friendly architecture using SML (boost.ext/SML or stateforward.SML). The design behaves like a pure actor model while remaining fully synchronous, run-to-completion (RTC), and queue-free.

## 1. Core Invariants
- **Run-To-Completion (RTC):** Dispatch is strictly synchronous. A top-level `process_event` must execute all internal and anonymous transitions to quiescence before returning.
- **No Queues:** Never use `process_queue`, `defer_queue`, mailboxes, or async "post-for-later" mechanisms.
- **Determinism:** Identical initial state + identical event sequence + identical payloads = identical execution paths and state changes.
- **Single-Writer:** Exactly one thread may execute inside an actor's `process_event` at any given time.
- **Zero Allocation:** Heap allocation is strictly forbidden during dispatch.
- **Bounded Work:** Every dispatch must have a provable upper bound on execution time and total transitions.

## 2. Events & Payloads
- **Trivial Types:** Events should be small, trivially copyable, and contain only immutable payloads. No owning pointers or dynamic containers (`std::string`, `std::vector`).
- **Pointer/Reference Validity:** Event payloads must remain valid for the entire top-level `process_event` lifecycle (including nested sync dispatch).
- **Internal Handoffs:** Use typed completions (`sml::completion<TEvent>`) to propagate event data between internal phases. Internal-only events may carry mutable references strictly for same-RTC handoff.
- **Dispatching:** Prefer compile-time typed dispatch. If runtime IDs are required, use static jump tables (`make_dispatch_table`) and validate ID bounds prior to indexing.

## 3. State & Context
- **Labels, Not Storage:** States act as control-flow labels. Store data in explicit, stable-address context objects injected via the state machine constructor.
- **Data Boundaries:** Context is for persistent, machine-owned state. Do not mirror ephemeral event payloads into context just to pass data between phases.
- **Error States:** Model error progression with explicit error states and events. Do not use context status fields as hidden control state.
- **Lock-Free Queries:** Use `visit_current_states` or `is(...)` for external observation. Do not require locks in the steady state.

## 4. Actions & Guards
- **Pure Guards:** Guards must be pure predicates returning `bool`. Context mutation is restricted to actions.
- **No Control-Flow Emulation:** Actions and their helpers must NOT contain runtime routing logic (`if`, `switch`, loops deciding execution paths, success/error/mode decisions). Moving logic to a helper does not make it compliant. All branching must be modeled as guarded transitions or explicit choice states.
- **Allowed Logic:** Compile-time conditionals (`#if`, `if constexpr`) and bounded, monotonic data-plane iteration loops are permitted in actions/helpers.
- **Execution Limits:** Actions must be short, non-blocking (no I/O, no mutex waits), and ideally `noexcept`. Split long-running tasks using "waiting" states.
- **Unexpected Events:** Catch unhandled external events using `sml::unexpected_event<...>`. Do not use wildcard `sml::_` guards that consume internal events.

## 5. Composition & Interaction
- **No Reentrancy:** An actor must never call its own `process_event` from within a guard or action. Model internal microflows with typed completions or acyclic anonymous transitions.
- **Synchronous Cross-Actor Calls:** Actor A may directly dispatch to Actor B synchronously, provided the call graph is acyclic and orchestrator-ordered.
- **Callbacks:** Synchronous callbacks (`emel::callback` style) are allowed for immediate, same-RTC notification. Never store callbacks in context for later execution.
- **Hierarchy:** Avoid ad-hoc shared base classes for machine wrappers. Submachines must obey RTC/no-queue rules. Orthogonal regions must not have conflicting side effects.

## 6. Time & Scheduling
- **External Time Only:** Actors must be driven by an external orchestrator. Inject time via event payloads (e.g., `tick{now_ns}`). Store deadlines in context and evaluate via guards.
- **Coroutines (`co_sm`):** Allowed by explicit opt-in only. Frame capacity must be pre-allocated. `co_await` must map exactly to SML phase boundaries (states/transitions) and must not hide inside guards, hot numeric loops, or routing helpers.

## 7. Pooling (`sm_pool`)
- **Usage:** Use `sml::utility::sm_pool` when a single router machine must drive a large pool of logical actors/items.
- **Storage:** Must be default-constructible or constructible from a pool size.
- **Dispatch:** Route by ID using `sml::utility::indexed_event`. Use contiguous batch APIs (`process_event_batch`). Validate IDs before indexing and avoid iterator overhead. Do not allocate during batch processing.

## 8. Style & Formatting
- **Naming:** `PascalCase` for public C++ types. `lower_snake_case` for internal types, states, and events. No macros in models.
- **Transitions:** Use destination-first syntax (`sml::state<dst> <= src + event [guard] / action`). Keep the destination and `<=` on the same line.
- **Tables:** Use leading-comma style for multi-row tables. Wrap complex tables in `// clang-format off` and `// clang-format on` to prevent auto-formatting degradation. Break large tables into visual sections.

## 9. Testing
- **Invariants:** Tests must explicitly assert determinism, bounded anonymous transitions, zero allocations (instrument global `new/delete`), and strict time budget compliance.
- **Validation:** Force states via `sml::testing::sm` in unit tests. Use a single-threaded orchestrator in integration tests to replay exact production dispatch orders.
