---
trigger: always_on
globs: "src/**/*, include/**/*, tests/**/*"
---

# C/C++ Real-Time Governance Rules

These rules govern code for real-time, deterministic, run-to-completion (RTC) actor systems.

## 1. Core Architecture & Determinism
* **Phase Separation**: Strictly separate **Initialization** (unbounded work, allocation allowed) from **Dispatch/RTC** (hard real-time, strict zero-allocation).
* **Determinism**: Given fixed inputs, initial state, and build, state transitions and outputs must be identical.
* **No Hidden State**: Never read wall-clock time, RNGs, filesystems, network, or mutable globals during Dispatch. Inject external data via event payloads.
* **Bounded Work**: Blocking I/O, paging, dynamic allocation, unbounded loops, recursion, and locks are strictly forbidden in Dispatch.
* **No Async in Dispatch**: Do not spawn threads, thread pools, or async tasks (`std::async`, coroutines) during Dispatch.

## 2. Compilation & Toolchain
* **Standards**: Require C17 (`-std=c17`) and C++20 (`-std=c++20`) or newer.
* **Strict Warnings**: Enable `-Wall -Wextra -Wpedantic` and `-Werror`. Do not suppress warnings without explicit justification.
* **Predictable Math**: Never use `-Ofast` or `-ffast-math` in deterministic builds.
* **No Exceptions**: Compile C++ with `-fno-exceptions`. All dispatch functions must be `noexcept`.

## 3. Memory & Allocations
* **Zero Heap in Dispatch**: `malloc`, `new`, `free`, `delete`, and STL container allocations are strictly forbidden in the Dispatch phase.
* **Pre-allocation**: Rely on fixed-capacity containers, stack allocations, and statically pre-allocated arenas.
* **Contiguous Data**: Prefer `std::array`, `std::span`, and fixed-capacity vectors over pointer-chasing structures like `std::list` or `std::map`.
* **Page Fault Prevention**: On hosted OS targets, lock memory (`mlockall`) and pre-fault the stack.
* **No Type Erasure**: Avoid `std::function` or `std::any` in hot paths due to hidden allocations. Use templates, function pointers, or fixed-capacity wrappers.

## 4. Concurrency & Synchronization
* **Single-Writer Invariant**: Only one thread may execute inside a given actor's state machine at a time. Do not re-enter recursively.
* **No Locks in Dispatch**: Mutexes and semaphores are forbidden. Rely on wait-free, single-writer architectures.
* **Explicit Atomics**: Use atomics for cross-thread data with explicit `std::memory_order`. Avoid default `seq_cst` unless justified.
* **Cache Alignment**: Pad and align frequently accessed atomics and hot data to cache-line boundaries (e.g., `alignas(64)`) to prevent false sharing.
* **Volatile**: Use `volatile` solely for MMIO registers. Never use it for inter-thread synchronization.

## 5. Safety & Undefined Behavior (UB)
* **Zero UB Tolerance**: Treat undefined behavior as a release-blocking defect.
* **No Signed Overflow**: Use checked, saturating, or widened arithmetic to prevent integer overflow.
* **No Type Punning**: Never use `reinterpret_cast` or `union` for type punning. Use `std::bit_cast` or `memcpy`.
* **Complete Initialization**: Initialize all structs entirely to avoid uninitialized padding reads.
* **CI Validation**: Mandate ASan, UBSan, and TSan in CI. Do not ship sanitizers in production.

## 6. Error Handling
* **Status Returns**: Propagate expected errors using status objects (e.g., `std::expected`) or error codes. Mark them `[[nodiscard]]`.
* **No Silent Failures**: Errors must be explicitly handled.
* **Fatal Violations**: On invariant failures, immediately enter a deterministic fault state or cleanly terminate (`abort()`). Do not attempt unbounded recovery in Dispatch.

## 7. APIs & ABI
* **Explicit Contracts**: Clearly label APIs as **RT-safe**, **Init-only**, or **Boundary**.
* **Bounded Parameters**: Pass memory views via `std::span` or `(ptr, len)`. Never pass owning containers by value in RT-safe APIs.
* **ABI Stability**: Use the C ABI (`extern "C"`) and standard-layout POD structs at boundaries. Do not expose STL types across shared libraries.
* **Explicit Units**: Make sizes, counts, and time units explicit in types (e.g., `std::chrono::nanoseconds`).

## 8. Observability & Logging
* **Lock-free Logging**: RT-safe logging must be non-blocking, allocation-free, and O(1) (e.g., lockless ring buffers with a deterministic overwrite/drop policy).
* **No String Formatting**: Do not use `printf` or string manipulation in hot loops. Log numeric event IDs and pre-formatted payloads.
* **Offline Formatting**: Defer aggregation, compression, and text formatting to non-RT boundary stages.

## 9. Hardware & System Boundaries
* **Syscalls**: Confine syscalls to Boundary/Orchestrator layers. Never invoke syscalls from within an actor's Dispatch phase.
* **Timers**: Manage timers externally. Deliver expirations to actors as standard synchronous events.
* **Interrupts (ISRs)**: Keep ISRs minimal. Communicate with RT threads using bounded shared state (e.g., atomic flags + latest-value buffers).
