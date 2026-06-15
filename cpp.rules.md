---
trigger: always_on
globs: "src/**/*, include/**/*, tests/**/*"
---

# C/C++ Real-Time Governance Rules

These rules govern code for real-time, deterministic, run-to-completion (RTC) actor systems.

- CPPRT-001: MUST strictly separate **Initialization** (unbounded work, allocation allowed) from **Dispatch/RTC** (hard real-time, strict zero-allocation).
- CPPRT-002: MUST guarantee that given fixed inputs, initial state, and build, state transitions and outputs are identical.
- CPPRT-003: MUST NOT read wall-clock time, RNGs, filesystems, network, or mutable globals during Dispatch. Inject external data only via event payloads.
- CPPRT-004: MUST forbid blocking I/O, paging, dynamic allocation, unbounded loops, recursion, and locks in Dispatch.
- CPPRT-005: MUST NOT spawn threads, thread pools, or async tasks (`std::async`, coroutines) during Dispatch.
- CPPRT-006: MUST require C17 (`-std=c17`) and C++20 (`-std=c++20`) or newer.
- CPPRT-007: MUST enable `-Wall -Wextra -Wpedantic` and `-Werror`. Do not suppress warnings without explicit justification.
- CPPRT-008: MUST NOT use `-Ofast` or `-ffast-math` in deterministic builds.
- CPPRT-009: MUST compile C++ with `-fno-exceptions`. All dispatch functions MUST be `noexcept`.
- CPPRT-010: MUST forbid `malloc`, `new`, `free`, `delete`, and STL container allocations in the Dispatch phase.
- CPPRT-011: MUST rely on fixed-capacity containers, stack allocations, and statically pre-allocated arenas.
- CPPRT-012: MUST prefer `std::array`, `std::span`, and fixed-capacity vectors over pointer-chasing structures like `std::list` or `std::map`.
- CPPRT-013: MUST lock memory (`mlockall`) and pre-fault the stack on hosted OS targets.
- CPPRT-014: MUST NOT use `std::function` or `std::any` in hot paths due to hidden allocations. Use templates, function pointers, or fixed-capacity wrappers.
- CPPRT-015: MUST ensure only one thread executes inside a given actor's state machine at a time. Do not re-enter recursively.
- CPPRT-016: MUST forbid mutexes and semaphores in Dispatch. Rely on wait-free, single-writer architectures.
- CPPRT-017: MUST use atomics for cross-thread data with explicit `std::memory_order`. Avoid default `seq_cst` unless justified.
- CPPRT-018: MUST pad and align frequently accessed atomics and hot data to cache-line boundaries (e.g., `alignas(64)`) to prevent false sharing.
- CPPRT-019: MUST use `volatile` solely for MMIO registers. Never use it for inter-thread synchronization.
- CPPRT-020: MUST treat undefined behavior as a release-blocking defect.
- CPPRT-021: MUST use checked, saturating, or widened arithmetic to prevent integer overflow.
- CPPRT-022: MUST NOT use `reinterpret_cast` or `union` for type punning. Use `std::bit_cast` or `memcpy`.
- CPPRT-023: MUST initialize all structs entirely to avoid uninitialized padding reads.
- CPPRT-024: MUST mandate ASan, UBSan, and TSan in CI. Do not ship sanitizers in production.
- CPPRT-025: MUST propagate expected errors using status objects (e.g., `std::expected`) or error codes. Mark them `[[nodiscard]]`.
- CPPRT-026: MUST explicitly handle errors. There must be no silent failures.
- CPPRT-027: MUST immediately enter a deterministic fault state or cleanly terminate (`abort()`) on invariant failures. Do not attempt unbounded recovery in Dispatch.
- CPPRT-028: MUST clearly label APIs as **RT-safe**, **Init-only**, or **Boundary**.
- CPPRT-029: MUST pass memory views via `std::span` or `(ptr, len)`. Never pass owning containers by value in RT-safe APIs.
- CPPRT-030: MUST use the C ABI (`extern "C"`) and standard-layout POD structs at boundaries. Do not expose STL types across shared libraries.
- CPPRT-031: MUST make sizes, counts, and time units explicit in types (e.g., `std::chrono::nanoseconds`).
- CPPRT-032: MUST ensure RT-safe logging is non-blocking, allocation-free, and O(1) (e.g., lockless ring buffers with a deterministic overwrite/drop policy).
- CPPRT-033: MUST NOT use `printf` or string manipulation in hot loops. Log numeric event IDs and pre-formatted payloads.
- CPPRT-034: MUST defer aggregation, compression, and text formatting to non-RT boundary stages.
- CPPRT-035: MUST confine syscalls to Boundary/Orchestrator layers. Never invoke syscalls from within an actor's Dispatch phase.
- CPPRT-036: MUST manage timers externally. Deliver expirations to actors as standard synchronous events.
- CPPRT-037: MUST keep ISRs minimal. Communicate with RT threads using bounded shared state (e.g., atomic flags + latest-value buffers).
