# C/C++ Real-Time Engineering Policy Rules (2025-2026)

> **Audience:** AI coding agents and human reviewers writing or maintaining C/C++ code for run-to-completion actor or state-machine systems with deterministic, bounded-latency requirements.
> 
> **Normative keywords:** MUST, MUST NOT, SHOULD, SHOULD NOT, MAY.

## 1. Scope and assumptions

- **CPPRT-001** MUST comply with the **RTC**, **no-queue**, **determinism**, **single-writer**, **no-heap-during-dispatch**, and **bounded-work** invariants defined in `sml.rules.md` for any actor or state-machine dispatch chain that follows this model.

- **CPPRT-002** MUST treat the following as **dispatch-critical** (real-time constrained) code paths:
  - `boost::sml::sm<...>::process_event(...)` execution, including guards, actions, entry/exit actions, and anonymous/internal transitions.
  - Any synchronous cross-actor call that occurs inside the above chain.

- **CPPRT-003** MUST separate execution into **Initialization/Configuration phase** (not time critical) and **Dispatch/RTC phase** (time critical).
  MUST document (in code comments or module docs) which functions are allowed in the Dispatch/RTC phase.

- **CPPRT-004** MUST define “deterministic” as: for a fixed build (compiler + flags), fixed hardware/OS configuration, and identical initial state and input event sequence (including payloads), the observed state transitions and side effects are identical.

- **CPPRT-005** MUST NOT read time, randomness, filesystem state, network state, environment variables, or global mutable process state directly from the Dispatch/RTC phase.
  MUST inject such information **only** via explicit event payloads or precomputed immutable configuration captured during Initialization.

- **CPPRT-006** MUST assume **hard real-time constraints** for rules that affect determinism and bounded latency (unless a rule explicitly says MAY/SHOULD for soft real-time).

- **CPPRT-007** MUST treat any operation with unbounded or input-dependent worst-case latency as forbidden in Dispatch/RTC (including blocking I/O, paging, locks, and dynamic allocation).

- **CPPRT-008** MUST NOT create background worker threads, task pools, or asynchronous jobs from within an actor’s Dispatch/RTC execution (aligns with “no message queue” and RTC invariants).

- **CPPRT-009** MUST use the terms **actor**, **dispatch**, **event**, **RTC chain**, and **dispatch-critical** consistently across the project and in reviews.

- **CPPRT-010** If a rule is violated for a specific subsystem, the violation MUST be:
  - explicitly documented near the code,
  - covered by a dedicated test that bounds the risk,
  - and tracked by an issue with an owner and removal plan.

## 2. Target standards (C17/C23, C++20/C++23) and compiler assumptions

- **CPPRT-011** MUST compile all C code as **C17** or newer (`-std=c17` or `-std=c23`).
  MUST compile all C++ code as **C++20** or newer (`-std=c++20` or `-std=c++23`).  
  MUST NOT rely on compiler extensions unless wrapped behind a portability layer and feature-tested.

- **CPPRT-012** MUST declare whether each target is **freestanding** or **hosted** (C/C++ standard terms) and MUST gate library usage accordingly (e.g., no `<iostream>` in freestanding builds).

- **CPPRT-013** MUST require a toolchain that correctly implements:
  - C11/C17 atomics (`<stdatomic.h>`) for C targets that use concurrency, or an explicit platform atomic layer.
  - C++11 atomics (`<atomic>`) for all C++ targets that use concurrency.

- **CPPRT-014** MUST compile with warnings enabled and treated as errors for production builds (`-Werror` or equivalent).
  MUST explicitly document any warning suppressions and keep them narrowly scoped.
  (Ref: GCC “-pedantic / -pedantic-errors” behavior: https://gcc.gnu.org/onlinedocs/gcc/Warnings-and-Errors.html)

- **CPPRT-015** SHOULD enable at least: `-Wall -Wextra -Wpedantic` (or MSVC equivalents) on all builds.
  SHOULD additionally enable hardening warnings appropriate for the project (e.g., `-Wconversion`, `-Wshadow`, `-Wdouble-promotion`) and fix violations rather than suppressing them.

- **CPPRT-016** MUST pin and version the compiler and standard library used in CI for each target triple.
  MUST record compiler version and key flags in build artifacts (e.g., `--version` output embedded into `--build-info`).

- **CPPRT-017** MUST define at least two build profiles:
  - **rt-debug** (instrumentation, sanitizers allowed, deterministic flags preserved)
  - **rt-release** (optimized, still deterministic, no sanitizer overhead)

- **CPPRT-018** MUST NOT use `-Ofast` or `-ffast-math` in Dispatch/RTC binaries that require deterministic numeric behavior, because they may change IEEE/ISO semantics.
  (Ref: GCC Optimize Options `-ffast-math`: https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html)

- **CPPRT-019** SHOULD use link-time dead stripping for embedded/code-size-constrained targets (`-ffunction-sections -fdata-sections` + linker GC) when supported.
  MUST verify that dead stripping does not remove required registration/entry points (no reliance on static initialization side effects).

- **CPPRT-020** MUST use compile-time feature detection (e.g., `__has_cpp_attribute`, `__cpp_lib_*`) for optional C++23 features (such as `<expected>`), and provide fallbacks.

- **CPPRT-021** SHOULD maintain a baseline flag set similar to:
  
  ```sh
  # C++ (rt-release)
  clang++ -std=c++20 -O2   -fno-exceptions -fno-rtti   -Wall -Wextra -Wpedantic -Werror   -Wconversion -Wshadow   -fvisibility=hidden
  ```
  
  Flags MUST be tailored per platform/toolchain and validated for jitter/throughput.

## 3. Determinism and undefined behavior rules

- **CPPRT-022** MUST treat **undefined behavior (UB)** as a release-blocking defect in all targets, including embedded and “performance-only” builds.

- **CPPRT-023** MUST run a CI configuration that executes tests with UB detection enabled (e.g., Clang/GCC UBSan).
  (Ref: UBSan overview: https://www.chromium.org/developers/testing/undefinedbehaviorsanitizer/ )

- **CPPRT-024** MUST NOT allow signed integer overflow in either C or C++ (it is UB).
  MUST use checked arithmetic, saturation arithmetic, or unsigned/explicitly widened arithmetic when overflow is possible.
  (Ref: SEI CERT C INT32-C: https://wiki.sei.cmu.edu/confluence/pages/viewpage.action?pageId=87152052)

- **CPPRT-025** MUST guard against UB in shifts and bit operations:
  - shift count MUST be in `[0, bit_width-1]`
  - left-shift on signed values that overflows MUST NOT occur
  - negative shift counts MUST NOT occur

- **CPPRT-026** MUST NOT write expressions whose correctness depends on unspecified/undefined order of evaluation or side effects (common in C/C++).
  (Ref: SEI CERT C EXP30-C “Do not depend on the order of evaluation for side effects”: https://abougouffa.github.io/awesome-coding-standards/sei-cert-c-2016.pdf)

- **CPPRT-027** MUST obey strict aliasing rules; code MUST be correct under `-O2` with strict aliasing enabled.
  MUST NOT “fix” aliasing violations by globally disabling aliasing (`-fno-strict-aliasing`) except as a temporary containment measure with a tracked bug.
  (Ref: GCC on strict-aliasing type punning warning: https://gcc.gnu.org/gcc-4.4/porting_to.html)

- **CPPRT-028** MUST NOT type-pun by `reinterpret_cast`/pointer-cast and dereference.
  MUST use `std::bit_cast` (C++20+) or `memcpy` for bit-level reinterpretation.
  (Ref: `std::bit_cast`: https://en.cppreference.com/w/cpp/numeric/bit_cast.html)

- **CPPRT-029** SHOULD use this pattern for bit reinterpretation:
  
  ```cpp
  #include <bit>
  #include <cstdint>
  
  std::uint32_t f32_bits(float x) noexcept {
    static_assert(sizeof(float) == sizeof(std::uint32_t));
    return std::bit_cast<std::uint32_t>(x);
  }
  ```

- **CPPRT-030** MUST NOT use `volatile` for inter-thread synchronization or atomicity.
  MUST use atomics or platform synchronization primitives instead.
  (Ref: C++ Core Guidelines CP.8: https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#cp8-dont-try-to-use-volatile-for-synchronization ; C volatile note: https://en.cppreference.com/w/c/language/volatile.html)

- **CPPRT-031** MAY use `volatile` only for:
  - memory-mapped I/O registers,
  - signal/ISR shared flags when paired with the platform’s required barriers,
  - and compiler-visibility constraints for special memory.
  MUST document the hardware contract at the declaration site.

- **CPPRT-032** MUST NOT access objects outside their lifetime (including after placement-new reuse) without following the C++ object lifetime rules.
  (Ref: lifetime rules: https://en.cppreference.com/w/cpp/language/lifetime.html)

- **CPPRT-033** If storage is reused via placement new for a different object lifetime, MUST use `std::launder` as required by the standard when reusing prior pointers.
  (Ref: `std::launder`: https://en.cppreference.com/w/cpp/utility/launder.html)

- **CPPRT-034** MUST treat data races as UB in C/C++ and as release-blocking defects.
  MUST ensure that any shared writable state is protected by the single-writer invariant or by correctly ordered atomics.

- **CPPRT-035** MUST NOT serialize/deserialize by `reinterpret_cast`ing structs or relying on padding/endianness.
  MUST use explicit serialization routines that define byte order and field widths.

- **CPPRT-036** MUST restrict `reinterpret_cast` (C++) and dangerous pointer casts (C/C++) to:
  - low-level boundary code (HAL, serialization, SIMD intrinsics wrappers),
  - with documented justification,
  - and with tests/sanitizer coverage.
  MUST NOT use casts to bypass the type system in general application logic.

- **CPPRT-037** When accessing object representations, MUST use `unsigned char`/`std::byte` pointers or `memcpy`, not incompatible typed pointers.

- **CPPRT-038** MUST NOT read uninitialized variables, padding, or storage.
  MUST initialize all fields explicitly, especially in structs that cross module boundaries.

- **CPPRT-039** MUST NOT perform pointer arithmetic that produces a pointer outside the bounds of the same object (except one-past-the-end as permitted).
  MUST avoid “pointer provenance” violations by using indices/offsets and bounds checks.

## 4. Memory model discipline (stack vs heap, static storage, alignment)

- **CPPRT-040** On hosted OS targets with virtual memory, MUST prevent page faults during Dispatch/RTC by:
  - locking memory where appropriate (`mlockall`/equivalent),
  - and pre-faulting stack/working set during Initialization.
  (Ref: `mlockall` note about reserving locked stack pages: https://man7.org/linux/man-pages/man2/mlock.2.html)

- **CPPRT-041** MUST set and enforce a per-thread stack budget for Dispatch/RTC threads (via linker script, RTOS config, or OS thread attributes).
  MUST measure worst-case stack usage for the dispatch-critical call chain and keep a safety margin.

- **CPPRT-042** MUST store dispatch-critical working state in:
  - actor-owned structs (typically embedded in the actor object),
  - stack allocations with statically known size,
  - or static storage with immutable or single-writer discipline.
  MUST NOT allocate from the heap in dispatch-critical code.

- **CPPRT-043** Any global/static object reachable from Dispatch/RTC MUST be **constant-initialized** (no dynamic initialization).
  MUST NOT rely on C++ dynamic initialization order across translation units.

- **CPPRT-044** MUST NOT use function-local `static` variables that require runtime initialization or guard checks inside Dispatch/RTC paths (may introduce hidden locks and nondeterministic first-use latency).

- **CPPRT-045** MUST explicitly align hot data structures (e.g., ring buffers, tensor tiles, atomic flags) to at least cache-line boundaries when false sharing or unaligned access can affect latency (`alignas(64)` or `std::hardware_destructive_interference_size` where supported).
  (Ref: `std::hardware_destructive_interference_size`: https://en.cppreference.com/w/cpp/thread/hardware_destructive_interference_size.html)

- **CPPRT-046** MUST ensure that frequently written variables from different threads do not share cache lines (padding or per-thread sharding).
  MUST review shared structs for false sharing risks as part of performance review.

- **CPPRT-047** MUST avoid accidental copies of large structs in Dispatch/RTC (pass by reference or `std::span`).
  MUST mark move/copy operations `=delete` or `noexcept` as appropriate to prevent unexpected copies.

- **CPPRT-048** MUST document the ownership model for every long-lived buffer:
  - owner (who allocates),
  - lifetime (when freed),
  - and access discipline (single-writer / read-only / atomic handoff).

- **CPPRT-049** Any pointer/reference stored in actor state and used across events MUST refer to storage with a stable address across the entire lifetime of that reference.
  If stability cannot be guaranteed (e.g., `std::vector` growth), MUST store indices/handles instead.

## 5. Allocation rules (global new/delete policy, arenas, fixed pools)

- **CPPRT-050** MUST actively enforce “no heap during dispatch” at runtime in debug/CI builds by trapping or counting allocations from `operator new`, `malloc`, and allocator backends while Dispatch/RTC is active.

- **CPPRT-051** For C++ Dispatch/RTC binaries, MUST define a project-wide policy for `operator new/delete`:
  - either globally disabled (`-fno-exceptions` + abort-on-new failure),
  - or redirected to a bounded allocator whose failure mode is deterministic.
  MUST NOT use the default system allocator from Dispatch/RTC.

- **CPPRT-052** SHOULD implement an allocation trap like:
  
  ```cpp
  #include <atomic>
  #include <new>
  #include <cstdlib>
  
  extern std::atomic<bool> g_dispatch_active;
  
  void* operator new(std::size_t n) {
    if (g_dispatch_active.load(std::memory_order_relaxed)) std::abort();
    if (void* p = std::malloc(n)) return p;
    std::abort();
  }
  void operator delete(void* p) noexcept { std::free(p); }
  ```
  
  (Replace `malloc/free` with your bounded allocator for production.)

- **CPPRT-053** MUST NOT call `malloc/calloc/realloc/free` or `new/delete` in Dispatch/RTC paths (guards/actions/entry/exit).
  MUST treat violations as correctness bugs, not “performance issues”.

- **CPPRT-054** If using arenas, MUST ensure arena capacity is fixed and sufficient for worst-case use.
  MUST define deterministic behavior on exhaustion (fail-fast, drop, or pre-agreed degradation) and MUST test it.

- **CPPRT-055** If using `std::pmr::monotonic_buffer_resource`, MUST ensure the upstream resource cannot fall back to heap allocation during Dispatch/RTC (size buffers accordingly and/or use a non-allocating upstream).
  (Ref: `monotonic_buffer_resource` can allocate from upstream when buffer is exhausted: https://en.cppreference.com/w/cpp/memory/monotonic_buffer_resource.html)

- **CPPRT-056** MAY use placement new only into explicitly owned storage (arena blocks, static buffers, or actor-owned raw storage).
  MUST pair with explicit destruction when required and MUST obey lifetime rules (`std::launder` where applicable).

- **CPPRT-057** MUST use fixed-capacity containers in Dispatch/RTC, or containers whose capacity is finalized during Initialization.
  For `std::vector`, MUST call `reserve(max)` during Initialization and MUST NOT perform operations that can increase capacity during dispatch.
  (Ref: vector reallocation invalidation: https://en.cppreference.com/w/cpp/container/vector/reserve.html)

- **CPPRT-058** MUST NOT build/append dynamic strings in Dispatch/RTC.
  MAY use `std::string_view`/`std::span<const char>` to refer to pre-existing stable storage, or use fixed-capacity string buffers.

- **CPPRT-059** MUST NOT use `std::function` in Dispatch/RTC hot paths (it may allocate and adds type-erasure overhead).
  MAY use templates, function pointers, or a fixed-buffer function wrapper with a compile-time size limit.
  (Ref: `std::function` stores a callable and may allocate; small buffer optimization is not guaranteed: https://en.cppreference.com/w/cpp/utility/functional/function.html)

- **CPPRT-060** MUST NOT use `<iostream>` in Dispatch/RTC (locale, formatting, and synchronization can allocate and/or lock).
  MAY use minimal, bounded formatting into preallocated buffers outside dispatch.

- **CPPRT-061** If exceptions are enabled in any build, MUST ensure Dispatch/RTC code cannot throw and cannot allocate exception objects.
  SHOULD compile production real-time binaries with exceptions disabled (see Exception Policy).

- **CPPRT-062** Any API that can allocate MUST either:
  - accept an explicit allocator/arena handle, or
  - be restricted to Initialization-only usage.
  MUST NOT hide allocations behind default parameters or global allocators.

- **CPPRT-063** SHOULD avoid long-lived heap allocations with many size classes on embedded/edge targets.
  If heap is used in Initialization, SHOULD prefer arenas/pools to reduce fragmentation.

## 6. Object lifetime and ownership discipline

- **CPPRT-064** MUST represent ownership explicitly:
  - C++: `std::unique_ptr` (or custom intrusive owner) for owning pointers.
  - C: explicit “create/destroy” API pairs or region allocation ownership.
  MUST NOT use owning raw pointers in new code.

- **CPPRT-065** MUST NOT use `std::shared_ptr` in Dispatch/RTC (atomic refcounting and potential control-block allocations can add jitter).
  MAY use shared ownership outside dispatch when lifecycle requires it.

- **CPPRT-066** Any borrowed pointer/reference used across events MUST point to storage whose lifetime exceeds the actor lifetime or is otherwise proven stable.
  MUST NOT store references to stack objects beyond the call where they are created.

- **CPPRT-067** MUST NOT return or store `std::string_view`/`std::span` pointing into temporary objects or containers that may reallocate.
  MUST ensure the referenced storage outlives the view.

- **CPPRT-068** In Dispatch/RTC, MUST avoid moves that can be linear-time (e.g., moving large vectors when allocator propagation triggers copies).
  MUST prefer fixed-size buffers or handles.

- **CPPRT-069** Event payload types used in SML dispatch SHOULD be trivially copyable and trivially destructible.
  MUST NOT embed ownership or dynamic containers in event payloads unless proven allocation-free and bounded.

- **CPPRT-070** Resources with expensive teardown (GPU handles, file descriptors, sockets) MUST be acquired and released outside Dispatch/RTC, and MUST be referenced by stable handles inside Dispatch/RTC.

- **CPPRT-071** MUST NOT use union type-punning for portability across compilers/ABIs.
  MUST use `memcpy`/`std::bit_cast` and explicit endianness conversion.

- **CPPRT-072** If pointer tagging is used (common in ML runtimes), it MUST be:
  - restricted to clearly documented bit patterns,
  - validated with static assertions on alignment,
  - and confined to a portability layer.

- **CPPRT-073** MUST ensure destructor work is bounded and deterministic.
  MUST NOT perform blocking I/O, locks, or heap frees in destructors that can run in Dispatch/RTC context.

## 7. Concurrency and threading rules (single-writer, no hidden locks)

- **CPPRT-074** MUST enforce the **single-writer invariant**: during any RTC chain, at most one thread executes inside a given actor’s state machine (`process_event`).
  MUST centralize dispatch through an orchestrator that enforces this (thread affinity or external serialization).

- **CPPRT-075** MUST NOT re-enter the same actor’s `process_event` recursively or via callbacks during its own dispatch (violates RTC bounded-work and complicates determinism).

- **CPPRT-076** MUST NOT use:
  - `std::async`, `std::future`, `std::promise`,
  - thread pools,
  - coroutines that suspend/resume asynchronously,
  in Dispatch/RTC paths.

- **CPPRT-077** MUST create threads only during Initialization (or an explicit lifecycle stage) and MUST NOT create or destroy threads from Dispatch/RTC.

- **CPPRT-078** MUST treat the following as **not Dispatch/RTC safe** unless proven otherwise:
  - heap allocators (internal locks),
  - locale/timezone APIs,
  - environment access (`getenv`),
  - iostreams,
  - `std::regex`,
  - dynamic loader APIs.

- **CPPRT-079** MUST NOT call blocking waits (`mutex::lock`, `condition_variable::wait`, `join`, `sleep`, blocking syscalls) in Dispatch/RTC.

- **CPPRT-080** Any cross-thread communication MUST:
  - be explicitly documented as a boundary,
  - be bounded (no unbounded buffering),
  - preserve determinism of actor state transitions.

- **CPPRT-081** MUST NOT implement actor mailboxes, deferred queues, or “post/emit later” systems for actor events.
  Asynchronous inputs MUST be converted into synchronous events by the orchestrator using a deterministic policy (drop/overwrite/backpressure), not by queueing.

- **CPPRT-082** SHOULD pin Dispatch/RTC threads to cores (or RTOS priorities) when jitter matters.
  MUST document the scheduling model (one actor per thread, actor groups per thread, etc.) and keep it stable.

- **CPPRT-083** SHOULD avoid thread-local storage reads/writes in Dispatch/RTC hot paths (can be cheap but may hinder portability and toolability).
  MUST NOT use TLS with dynamic initialization in Dispatch/RTC.

- **CPPRT-084** SHOULD use concurrency analyzers where applicable (TSan in CI for hosted targets; Clang Thread Safety Analysis annotations if locks are used).
  (Ref: Clang Thread Safety Analysis: https://clang.llvm.org/docs/ThreadSafetyAnalysis.html)

- **CPPRT-085** If a hosted OS real-time scheduler is used, MUST design to avoid priority inversion (prefer no locks; otherwise use priority inheritance/protection as required by the platform).
  (Ref: POSIX priority inheritance: https://pubs.opengroup.org/onlinepubs/7908799/xsh/pthread_mutexattr_setprotocol.html)

## 8. Locking and synchronization constraints (if allowed)

- **CPPRT-086** MUST treat locks as forbidden in Dispatch/RTC by default (mutexes, RW locks, semaphores, futexes, OS critical sections).

- **CPPRT-087** If locks are required, they MUST be confined to explicit boundary modules (I/O, OS integration) and MUST NOT be used inside SML guards/actions/entry/exit.

- **CPPRT-088** If a lock must be used in any time-critical thread, MUST use a non-blocking acquisition strategy (`try_lock` with bounded retries) or a design that eliminates contention.
  MUST define and test the worst-case bound.

- **CPPRT-089** On platforms with priority scheduling, any unavoidable mutex in a real-time thread MUST use priority inheritance or priority ceiling/protection where supported.
  (Ref: priority inheritance semantics: https://man7.org/linux/man-pages/man3/pthread_mutexattr_getprotocol.3p.html)

- **CPPRT-090** If more than one lock exists in the system, MUST define a global lock order and MUST enforce it (static analysis or code review checklists).
  MUST NOT acquire locks out of order.

- **CPPRT-091** Any allocator reachable from Dispatch/RTC MUST be lock-free or single-thread confined by construction.
  MUST NOT call the system allocator from Dispatch/RTC because it typically uses internal locks.

- **CPPRT-092** Logging/tracing performed in Dispatch/RTC MUST NOT acquire locks; it MUST be non-blocking and bounded (see Logging rules).

- **CPPRT-093** MUST NOT use lock-based memory reclamation patterns in Dispatch/RTC hot paths (e.g., global freelists with mutexes).
  MUST use per-actor or per-thread pools, or non-blocking designs with bounded work.

## 9. Atomic usage and memory ordering rules

- **CPPRT-094** MUST use atomics only where data crosses threads/cores/ISRs.
  Within a single-writer actor, MUST prefer plain non-atomic loads/stores.

- **CPPRT-095** In C++ code, MUST explicitly specify `std::memory_order` for every atomic `load/store/exchange/compare_exchange` in real-time components.
  MUST NOT rely on default `seq_cst` implicitly.
  (Ref: `std::memory_order`: https://en.cppreference.com/w/cpp/atomic/memory_order.html)

- **CPPRT-096** MAY use `memory_order_relaxed` for independent counters/statistics that do not publish data and do not participate in synchronization.

- **CPPRT-097** MUST use **release** on the publishing side and **acquire** on the consuming side for ownership/data handoff patterns.
  MUST document the handoff variable and the data it protects.

- **CPPRT-098** SHOULD avoid `memory_order_seq_cst` in hot paths unless a global total order is required and justified; it can add fences and jitter.

- **CPPRT-099** MUST NOT use `atomic_thread_fence` unless implementing a documented low-level pattern.
  Any fence usage MUST include a comment that states the synchronization contract.
  (Ref: `std::atomic_thread_fence`: https://en.cppreference.com/w/cpp/atomic/atomic_thread_fence.html)

- **CPPRT-100** MUST NOT use atomic operations on bit-fields or rely on bit-field layout across compilers; use full-width atomics instead.

- **CPPRT-101** MUST align and pad atomics that are written frequently or by different threads to avoid false sharing.
  SHOULD co-locate read-mostly atomics separately from write-heavy atomics.

- **CPPRT-102** MUST NOT access the same memory location sometimes atomically and sometimes non-atomically across threads (data race/UB).
  MUST encapsulate the variable behind a single access API.

- **CPPRT-103** MUST handle spurious failures correctly (`compare_exchange_weak` in a loop).
  MUST use `compare_exchange_strong` only when necessary.

- **CPPRT-104** If using C++20 atomic wait/notify, MUST confine it to boundary threads (not actor Dispatch/RTC) and MUST bound waiting time.

- **CPPRT-105** If an ISR writes data read by a Dispatch/RTC thread, MUST use:
  - atomic flags for “data ready” publication,
  - and a deterministic overwrite/drop policy (no unbounded buffering).
  MUST NOT perform dynamic allocation or locking in the ISR.

## 10. Exception policy (C++ only) — required stance

- **CPPRT-106** MUST compile real-time/dispatch-critical C++ components with exceptions **disabled** (`-fno-exceptions` or equivalent) unless an explicit exception is granted for a boundary-only module.

- **CPPRT-107** All functions that may execute during Dispatch/RTC (guards/actions/entry/exit and anything they call) MUST be `noexcept` (directly or effectively).
  MUST treat any potential throw path as a defect.

- **CPPRT-108** MUST NOT use exceptions for control flow, retries, or expected failure in any part of the system.

- **CPPRT-109** If any linked code can throw (third-party libs, non-RT subsystems), MUST define an explicit boundary wrapper that:
  - catches all exceptions,
  - converts to a deterministic error signal,
  - and prevents exceptions from propagating into actor Dispatch/RTC.

- **CPPRT-110** MUST ensure destructors are `noexcept` (the default in modern C++), and MUST NOT throw from destructors.

- **CPPRT-111** SHOULD note in project docs that exceptions can be unacceptable in life-critical hard-real-time code due to control-flow and unwind-cost unpredictability.
  (Ref: C++ Core Guidelines resource management note: https://cpp-core-guidelines-docs.vercel.app/resource)

## 11. Error handling model (error codes vs status objects vs exceptions)

- **CPPRT-112** MUST represent recoverable/expected errors without exceptions:
  - C: return error codes or status structs.
  - C++: return status objects (e.g., `std::expected<T,E>` in C++23) or error codes.
  (Ref: `std::expected` (C++23): https://en.cppreference.com/w/cpp/utility/expected.html)

- **CPPRT-113** Error/status return types used in Dispatch/RTC MUST be trivially copyable/movable and MUST NOT allocate.
  MUST avoid storing strings or dynamic data in error objects in Dispatch/RTC.

- **CPPRT-114** MUST mark fallible functions as `[[nodiscard]]` (C++), or enforce via static analysis in C, so errors are not silently ignored.

- **CPPRT-115** MUST propagate errors explicitly through return values, not via global state or hidden thread-local state.
  MUST keep error propagation bounded (no retries with unbounded loops in Dispatch/RTC).

- **CPPRT-116** MUST classify failures into:
  - **recoverable** (modeled as state transitions / error returns),
  - **fatal** (leads to deterministic fail-stop or fault state),
  - **external** (I/O/service down; handled outside dispatch).
  MUST document classification for each module boundary.

- **CPPRT-117** On fatal invariant violation, MUST take a deterministic action:
  - enter a defined “fault” state and stop processing further events, OR
  - terminate the process with a clear diagnostic.
  MUST NOT attempt best-effort recovery in Dispatch/RTC after invariant failure.

- **CPPRT-118** MUST NOT rely on `errno` in Dispatch/RTC code.
  If interacting with syscalls, MUST capture error codes at the boundary and convert to stable domain errors before dispatch.

- **CPPRT-119** MUST ensure error creation/propagation does not implicitly log or allocate in Dispatch/RTC.
  Logging on error MUST follow bounded logging rules.

- **CPPRT-120** In C++, MAY use `[[noreturn]]` and `std::terminate`/`abort` for fatal paths.
  MUST ensure any `noreturn` path does not perform unbounded work.

- **CPPRT-121** MUST standardize the project’s error conventions (success code, error enum ranges, and mapping to OS errors).
  MUST NOT create ad-hoc error code meanings per module.

## 12. API surface design for real-time systems

- **CPPRT-122** MUST clearly label APIs as one of:
  - **RT-safe** (may be called in Dispatch/RTC),
  - **Init-only** (must not be called in Dispatch/RTC),
  - **Boundary** (I/O/OS integration; called outside Dispatch/RTC).
  This labeling MUST be visible in headers (comments, attributes, or naming conventions).

- **CPPRT-123** Any API that may allocate MUST state so in its documentation and MUST NOT be callable from RT-safe contexts.
  RT-safe APIs MUST be allocation-free by construction.

- **CPPRT-124** RT-safe APIs MUST NOT block, wait, sleep, or perform syscalls with unpredictable latency.

- **CPPRT-125** RT-safe APIs SHOULD accept `std::span<T>`/`std::span<const T>` (C++20) or `(ptr,len)` pairs (C) for buffer parameters.
  MUST NOT accept owning containers as inputs in RT-safe APIs.
  (Ref: `std::span`: https://en.cppreference.com/w/cpp/container/span.html)

- **CPPRT-126** MUST make sizes, counts, and time units explicit in API types (e.g., `std::chrono::nanoseconds`, `size_t`).
  MUST NOT pass “raw ints” with ambiguous units.

- **CPPRT-127** APIs MUST not expose partially initialized objects.
  Constructors/factories MUST return fully initialized objects or an error status (no half-valid instances).

- **CPPRT-128** Actor boundary APIs (event ingress/egress) MUST use stable POD-like event types and MUST avoid ABI-unstable STL types across shared-library boundaries.

- **CPPRT-129** MUST NOT accept callbacks in RT-safe APIs that can re-enter actor dispatch or call back into unknown code during Dispatch/RTC (breaks bounded-work reasoning).

- **CPPRT-130** MUST model long-running operations as explicit state transitions driven by events, not as blocking calls or internal background work.

- **CPPRT-131** Public APIs MUST include a deprecation mechanism (compile-time attribute or macro) and a removal policy.
  MUST NOT silently change semantics of RT-safe APIs without versioning.

## 13. ABI and binary layout discipline

- **CPPRT-132** MUST NOT expose C++ standard library types (e.g., `std::string`, `std::vector`, `std::expected`) in stable ABI boundaries between separately compiled/shared components.
  MUST use C ABI (`extern "C"`) and POD structs for stable interfaces.

- **CPPRT-133** SHOULD compile libraries with hidden symbol visibility by default and explicitly export the public API surface (`-fvisibility=hidden` + export macros on ELF platforms).

- **CPPRT-134** Any struct used in binary interfaces MUST be:
  - `std::is_standard_layout_v == true`,
  - `std::is_trivially_copyable_v == true`,
  - and validated with `static_assert` on size and offsets where relevant.

- **CPPRT-135** MUST define endianness for every serialized/binary format.
  MUST use explicit byte-order conversions; MUST NOT assume host endianness.

- **CPPRT-136** MUST NOT use `#pragma pack`/`__attribute__((packed))` for performance-critical data that is frequently accessed; it can cause misaligned loads and UB on some targets.
  MAY use packed structs only for wire parsing, followed immediately by explicit decoding into aligned native structs.

- **CPPRT-137** MUST `static_assert(alignof(T) >= N)` when code relies on alignment (SIMD loads, pointer tagging).
  MUST provide fallback paths when alignment cannot be guaranteed.

- **CPPRT-138** MUST avoid One Definition Rule violations across translation units and packages (common in header-only template code).
  MUST centralize configuration macros in a single header and keep them consistent.

- **CPPRT-139** If providing a C ABI, MUST use stable integer/enum error codes and MUST document them as part of the ABI contract.

- **CPPRT-140** MUST initialize padding bytes in structs that cross trust boundaries or are hashed/serialized, to avoid information leaks and nondeterministic comparisons.

- **CPPRT-141** For libraries with stable ABI, MUST maintain ABI compatibility tests (e.g., size/layout checks and symbol checks) across releases.

## 14. Data structures for high throughput (contiguous memory, cache locality)

- **CPPRT-142** MUST prefer contiguous storage (`std::array`, `std::span`, fixed-capacity vectors, SoA layouts) for hot-path data.
  MUST justify pointer-chasing structures in performance-critical code.

- **CPPRT-143** MUST NOT use `std::list`, intrusive lists, or general linked lists in Dispatch/RTC hot paths unless a benchmark proves it is superior and work is bounded.

- **CPPRT-144** If `std::vector` is used in RT-safe code, its capacity MUST be fixed during Initialization (`reserve`) and MUST NOT grow in Dispatch/RTC.
  MUST treat any reallocation as a correctness violation.

- **CPPRT-145** MUST NOT use `std::unordered_map`/`std::unordered_set` in Dispatch/RTC unless:
  - the allocator is fixed/pool-based,
  - rehashing is impossible (reserved and load-factor bounded),
  - and worst-case probe bounds are acceptable and tested.

- **CPPRT-146** SHOULD use sorted vectors (“flat_map” style) for small or mostly-static keyspaces to improve cache locality and predictability.

- **CPPRT-147** For CPU-bound ML engines, SHOULD prefer Structure-of-Arrays (SoA) or blocked/tiled layouts for tensor metadata and hot activation buffers to improve cache utilization and SIMD friendliness.

- **CPPRT-148** MUST avoid implicit buffer copies in API boundaries (return-by-value of large buffers, passing large structs by value).
  MUST pass by `span` or reference and keep ownership explicit.

- **CPPRT-149** If ring buffers or producer/consumer buffers are used for logging or boundary telemetry, MUST place producer and consumer indices on separate cache lines to avoid false sharing.
  (Ref: Linux tracing ring buffer design emphasizes lockless/bounded behavior: https://docs.kernel.org/trace/ring-buffer-design.html)

- **CPPRT-150** MUST avoid allocator-heavy abstractions (polymorphic allocators with upstream heap, node-based containers) in hot paths unless bounded and proven allocation-free.

- **CPPRT-151** MUST define memory budgets for:
  - actor state,
  - per-thread scratch buffers,
  - and per-subsystem arenas,
  and MUST fail deterministically if budgets are exceeded.

- **CPPRT-152** MAY use explicit prefetch intrinsics only when profiling demonstrates benefit.
  MUST keep prefetch usage behind a portability macro and benchmark across target CPUs.

- **CPPRT-153** MUST NOT perform dynamic formatting (printf-like formatting) inside hot loops; log numeric codes or sample data into preallocated buffers.

- **CPPRT-154** In hot paths, SHOULD minimize unpredictable branches (data-dependent conditionals) and SHOULD prefer data layouts that improve predictability.
  MUST justify manual branch prediction hints (`__builtin_expect`) with profiling evidence.

## 15. Avoiding dynamic polymorphism in hot paths

- **CPPRT-155** MUST NOT use virtual function calls in Dispatch/RTC hot loops where bounded latency matters.
  MUST justify any virtual dispatch with profiling evidence and boundedness arguments.

- **CPPRT-156** MUST NOT use RTTI (`dynamic_cast`, `typeid`) in Dispatch/RTC.
  MAY use RTTI only in Initialization, diagnostics, or tooling builds.

- **CPPRT-157** SHOULD prefer templates, CRTP, or `std::variant` visitation for polymorphism in hot paths.
  MUST keep variant alternatives bounded and known at compile time.

- **CPPRT-158** MUST avoid general type erasure (`std::any`, `std::function`, `std::move_only_function`) in RT-critical paths unless the implementation is proven allocation-free and bounded.

- **CPPRT-159** MUST NOT rely on dynamic loader plugins (dlopen/loadlibrary) or late-binding function resolution in Dispatch/RTC.
  All function pointers used in Dispatch/RTC MUST be resolved during Initialization.

- **CPPRT-160** MUST NOT delete through a base pointer in Dispatch/RTC (dynamic allocation and virtual destructors are forbidden there).
  Object graphs MUST have deterministic lifetimes managed outside dispatch.

- **CPPRT-161** If using `std::variant`, MUST keep visitation logic simple and MUST avoid nested variants that can cause combinatorial visitation code growth in hot paths.

- **CPPRT-162** MAY use explicit vtable structs (manual function pointer tables) for ABI-stable hot-path polymorphism, but MUST ensure:
  - tables are immutable,
  - calls are bounded,
  - and initialization happens before Dispatch/RTC.

## 16. Compile-time vs runtime tradeoffs (constexpr, templates, codegen)

- **CPPRT-163** MUST prefer compile-time validation (`static_assert`, `constexpr` checks) for invariants that can be proven at compile time (sizes, ranges, alignment, enum completeness).

- **CPPRT-164** Any lookup table used in Dispatch/RTC SHOULD be `constexpr` and stored in read-only memory.
  MUST NOT lazily initialize such tables at first use inside dispatch.

- **CPPRT-165** MUST keep template recursion and compile-time computation bounded to avoid runaway compile times and code size.
  MUST set and enforce build time budgets in CI for large monorepos.

- **CPPRT-166** MUST NOT generate code or JIT at runtime in Dispatch/RTC.
  If JIT/codegen is required for ML engines, it MUST occur during Initialization with deterministic inputs.

- **CPPRT-167** SHOULD use explicit template instantiation in `.cc/.cpp` files for heavy templates to reduce compile times and improve link determinism.

- **CPPRT-168** MUST NOT rely on constexpr evaluation that differs across compilers (non-portable intrinsics).
  MUST keep constexpr logic within standard-defined behavior.

- **CPPRT-169** If using code generation (protobuf, flatbuffers, custom), MUST pin generator versions and MUST treat generated code as part of the reproducible build contract.

- **CPPRT-170** MUST keep compile-time heavy components (templates, generated code) behind clear module boundaries to prevent cascading rebuilds.

## 17. Inline, LTO, and optimization guidance

- **CPPRT-171** MUST choose optimization levels (`-O2` vs `-O3`) based on measured throughput and latency jitter on target hardware.
  MUST keep a documented baseline per target.

- **CPPRT-172** MUST NOT use `-Ofast` or `-ffast-math` for deterministic real-time builds (can change floating-point semantics and break reproducibility).
  (Ref: GCC Optimize Options: https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html)

- **CPPRT-173** If bitwise-deterministic floating point results are required across CPUs, MUST control FP contraction and reassociation (e.g., disable contraction where needed) and MUST document the chosen policy per target.

- **CPPRT-174** MAY enable LTO for throughput, but MUST:
  - measure its impact on latency/jitter,
  - keep the link step deterministic (pinned toolchain),
  - and ensure debug symbol strategy is workable for incident response.

- **CPPRT-175** MAY enable PGO for throughput, but MUST:
  - collect profiles on representative workloads,
  - treat profile data as a versioned build input,
  - and measure worst-case latency effects (branch layout changes can affect i-cache behavior).

- **CPPRT-176** MUST avoid `always_inline` abuse.
  MAY use `[[gnu::always_inline]]`/`__forceinline` only when profiling proves benefit and code size impact is acceptable.

- **CPPRT-177** SHOULD keep frame pointers in at least one shipping profile for production observability if it does not violate latency budgets (`-fno-omit-frame-pointer` on many targets).

- **CPPRT-178** MUST NOT rely on flags that change language semantics to “hide” UB (e.g., `-fwrapv`) as a general policy.
  If used for legacy compatibility, MUST still treat overflow/UB as defects and plan removal.

- **CPPRT-179** On hosted ELF targets, SHOULD enable standard linker hardening flags (RELRO, NOW, PIE) unless they conflict with real-time constraints; any disablement MUST be documented.

- **CPPRT-180** MUST ensure that build outputs do not depend on:
  - absolute paths in debug info (unless stripped),
  - timestamps (use reproducible build flags where available),
  - or non-versioned generated sources.

## 18. Logging and observability rules (bounded, no allocation)

- **CPPRT-181** Logging/tracing callable from Dispatch/RTC MUST be:
  - non-blocking,
  - allocation-free,
  - and bounded O(1) per call.

- **CPPRT-182** MUST NOT perform syscalls or device I/O (write to files, sockets, stdout) from Dispatch/RTC logging.
  MUST buffer logs into preallocated memory and flush outside Dispatch/RTC.

- **CPPRT-183** SHOULD implement RT-safe logging as a fixed-size ring buffer with overwrite-or-drop behavior on overflow.
  (Ref: Linux kernel lockless ring buffer supports overwrite/producer-consumer modes: https://docs.kernel.org/trace/ring-buffer-design.html)

- **CPPRT-184** MUST define a deterministic overflow policy for log buffers (drop newest, drop oldest, overwrite oldest) and MUST test it.

- **CPPRT-185** MUST NOT make functional decisions based on whether logging succeeds (e.g., “if log buffer full then change behavior”) in Dispatch/RTC.
  Logging is observability only.

- **CPPRT-186** MUST avoid string formatting in Dispatch/RTC.
  SHOULD log numeric event IDs + small fixed payloads, or format later during offline decode.

- **CPPRT-187** If using `std::source_location` for diagnostics, MUST ensure it does not leak absolute paths into production artifacts and MUST keep it out of Dispatch/RTC hot paths unless proven bounded.
  (Ref: `std::source_location`: https://en.cppreference.com/w/cpp/utility/source_location.html)

- **CPPRT-188** On embedded targets, MAY use a non-blocking real-time transfer mechanism (e.g., SEGGER RTT) provided it is bounded and does not allocate.
  (Ref: SEGGER RTT design goal is real-time transfer without halting the CPU: https://www.segger.com/products/debug-probes/j-link/technology/about-real-time-transfer/ )

- **CPPRT-189** MUST compile-time gate trace points (`#if TRACE_ENABLED`) to allow zero-overhead removal in production profiles when required.

- **CPPRT-190** MUST perform any aggregation, compression, encoding, or export of telemetry outside Dispatch/RTC in a bounded, scheduled boundary stage.

## 19. Time sources and scheduling discipline

- **CPPRT-191** MUST NOT read wall-clock or monotonic time directly inside Dispatch/RTC.
  MUST obtain time in the orchestrator/boundary layer and inject it into the actor as part of an event payload when needed for decisions.

- **CPPRT-192** Boundary/orchestrator code MUST use a monotonic clock for measuring durations (not `CLOCK_REALTIME`).
  On Linux, SHOULD use `CLOCK_MONOTONIC` or `CLOCK_MONOTONIC_RAW` depending on your NTP/adjustment requirements.
  (Ref: `clock_gettime` clock IDs: https://man7.org/linux/man-pages/man2/clock_gettime.2.html)

- **CPPRT-193** On hosted OS targets, MUST explicitly configure scheduling policy/priority for Dispatch/RTC threads (where supported) and MUST test behavior under load.
  (Ref: Linux scheduling overview: https://man7.org/linux/man-pages/man7/sched.7.html)

- **CPPRT-194** MUST NOT call `sleep`, `nanosleep`, `sched_yield`, or equivalent from Dispatch/RTC.
  Any waiting MUST be moved to the orchestrator and expressed as future events/timers.

- **CPPRT-195** MUST centralize timer management in a single boundary module.
  Actors MUST receive timer expirations as explicit events and MUST NOT arm OS timers from Dispatch/RTC.

- **CPPRT-196** If any blocking primitives exist in boundary code, MUST ensure they do not introduce priority inversion for real-time threads (priority inheritance/protection as appropriate).
  (Ref: POSIX mutex protocol attributes: https://pubs.opengroup.org/onlinepubs/7908799/xsh/pthread_mutexattr_setprotocol.html)

- **CPPRT-197** If using memory locking to avoid page faults, MUST do so before entering the real-time section and MUST pre-touch stack/heap pages needed for Dispatch/RTC.
  (Ref: `mlockall` stack prefault advice: https://manpages.ubuntu.com/manpages/jammy/man2/mlock.2.html)

- **CPPRT-198** MUST treat syscalls as boundary-only operations.
  Actors MUST not call syscalls from Dispatch/RTC (file/network/clock/scheduler APIs).

- **CPPRT-199** MUST define per-event and per-actor time budgets.
  MUST measure and alert/abort deterministically on budget violations (e.g., watchdog event that triggers a fault state).

- **CPPRT-200** MUST test timer/event behavior under CPU saturation and I/O pressure to ensure deadlines and ordering remain within spec.

## 20. Interfacing with hardware / syscalls safely

- **CPPRT-201** MUST perform syscalls only in boundary/orchestrator code (I/O, scheduling, memory management).
  MUST NOT call syscalls from within actor Dispatch/RTC.

- **CPPRT-202** Boundary I/O MUST be configured as non-blocking where feasible.
  MUST bound per-iteration work when polling I/O (no unbounded draining loops).

- **CPPRT-203** MUST NOT allocate, lock, or perform heavy work in interrupt context.
  ISR code MUST be minimal and MUST communicate with the orchestrator/actor via bounded shared state (latest-value + atomic flag).

- **CPPRT-204** MUST isolate memory-mapped I/O access behind a dedicated HAL (hardware abstraction layer).
  MUST NOT scatter `volatile` register accesses throughout actor logic.

- **CPPRT-205** For MMIO registers, MUST use `volatile` qualified access as required to prevent elision, and MUST apply the platform’s required memory barriers/fences for ordering with devices.
  MUST document the ordering requirements per device.

- **CPPRT-206** MUST NOT perform misaligned loads/stores via pointer casts.
  MUST use `memcpy` or architecture-provided unaligned access helpers when decoding packed wire/device formats.

- **CPPRT-207** MUST avoid Unix signals and signal handlers in Dispatch/RTC threads (asynchronous and hard to bound).
  If signals are required, MUST handle them in a dedicated boundary thread.

- **CPPRT-208** MUST wrap OS/hardware APIs behind thin adapters that:
  - translate errors to project status types,
  - normalize time units,
  - and make blocking/allocating behavior explicit.

- **CPPRT-209** MUST bound any device-driver interaction loops (e.g., draining RX descriptors) and MUST enforce maximum work per dispatch tick.

- **CPPRT-210** MUST NOT load/unload shared libraries or resolve symbols dynamically in Dispatch/RTC.
  If dynamic loading is required, MUST do it during Initialization only.

## 21. Security and safety constraints (UB avoidance, bounds, sanitizers)

- **CPPRT-211** For hosted targets, MUST run CI jobs with:
  - AddressSanitizer (ASan),
  - UndefinedBehaviorSanitizer (UBSan),
  - and ThreadSanitizer (TSan) where concurrency exists,
  on representative test suites.
  (Ref: Clang sanitizer documentation: https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html)

- **CPPRT-212** MUST NOT ship sanitizers in production real-time binaries unless explicitly justified (they change timing and may allocate).
  Sanitizers are for CI/testing.

- **CPPRT-213** MUST gate merges on static analysis for safety-critical code:
  - clang-tidy (bugprone/performance/readability),
  - and at least one additional analyzer (Clang Static Analyzer, GCC -fanalyzer, or commercial tool).
  (Ref: clang-tidy docs: https://clang.llvm.org/extra/clang-tidy/ ; Clang Static Analyzer: https://clang.llvm.org/docs/ClangStaticAnalyzer.html)

- **CPPRT-214** SHOULD align coding rules with an established secure/safety standard (CERT C/C++, MISRA C/C++ or AUTOSAR C++) and MUST document deviations.
  (Ref: Cppcheck MISRA addon notes: https://cppcheck.sourceforge.io/manual.html)

- **CPPRT-215** MUST validate all untrusted inputs at trust boundaries (network packets, files, IPC, hardware DMA buffers).
  Parsing/validation MUST be separated from Dispatch/RTC logic and MUST produce validated, bounded data structures before dispatch.

- **CPPRT-216** Bit-level and SIMD code MUST be reviewed for UB (alignment, aliasing, overflow, shifts).
  MUST include targeted tests and UBSan coverage for these modules.

- **CPPRT-217** MUST perform explicit, checked conversions between integer sizes and signedness.
  MUST NOT rely on implicit narrowing conversions.

- **CPPRT-218** On hosted targets, SHOULD enable stack protection and fortify features (`-fstack-protector-strong`, `_FORTIFY_SOURCE`) where supported and compatible with latency requirements.
  Any disablement MUST be documented with rationale.

- **CPPRT-219** MUST NOT store credentials, private keys, or secrets in source control.
  MUST scan repositories for secrets in CI.

- **CPPRT-220** MUST pin toolchain and third-party dependency versions (including compiler, libc++, libstdc++, and embedded SDKs) for reproducible, auditable builds.

- **CPPRT-221** SHOULD fuzz parsers/decoders and boundary protocol handling (libFuzzer/AFL++/Honggfuzz) in CI for hosted targets.
  MUST keep fuzz targets separate from Dispatch/RTC binaries.

- **CPPRT-222** MUST initialize memory before use (no uninitialized reads).
  MUST avoid “use of uninitialized padding” in comparisons/hashes (zero padding or use field-wise operations).

## 22. Testing and verification strategy for real-time code

- **CPPRT-223** MUST have automated tests that replay recorded event sequences and assert identical:
  - state transitions,
  - actor outputs,
  - and (when enabled) trace events.

- **CPPRT-224** MUST have tests that verify **zero dynamic allocation** occurs during Dispatch/RTC by:
  - trapping `new/malloc`,
  - or using allocator instrumentation,
  - and failing the test on any allocation.

- **CPPRT-225** MUST have tests that assert per-event upper bounds on:
  - number of transitions/actions,
  - loop iterations in hot paths,
  - and maximum batch sizes processed per dispatch tick.

- **CPPRT-226** SHOULD run latency/jitter microbenchmarks on representative hardware in CI or nightly jobs.
  MUST treat regressions beyond a defined threshold as failures requiring triage.

- **CPPRT-227** If multi-threading exists at boundaries, MUST run ThreadSanitizer (hosted targets) and stress tests to detect data races and ordering bugs.

- **CPPRT-228** MUST keep fuzzing/sanitizer-heavy test binaries separate from production RT binaries.
  MUST ensure fuzz targets cover boundary parsing/decoding logic.

- **CPPRT-229** MUST run clang-tidy/cppcheck/static analyzer as part of CI and MUST treat new findings in RT-critical modules as failures.

- **CPPRT-230** For embedded targets, SHOULD run hardware-in-the-loop tests that validate timing, scheduling, and device interactions under load.

- **CPPRT-231** MUST include fault-injection tests for boundary failures (I/O errors, corrupted inputs, allocation exhaustion during init) and verify deterministic error handling.

- **CPPRT-232** MUST define coverage expectations (statement/branch) for safety-critical code and MUST ensure coverage is measured in CI for hosted targets where feasible.

- **CPPRT-233** If using golden output traces, MUST ensure they are stable across platforms/builds for the supported targets, or MUST scope them to a specific platform/toolchain.

- **CPPRT-234** Performance profiling runs MUST be repeatable:
  - pinned CPU frequency governors (hosted OS),
  - fixed thread affinity,
  - and controlled background load.
  MUST document the profiling protocol.

## 23. Anti-patterns list (explicit “DO NOT” rules)

- **CPPRT-235** MUST NOT allocate from heap (`new`, `malloc`, container growth) anywhere in Dispatch/RTC.

- **CPPRT-236** MUST NOT call blocking waits (`mutex::lock`, `cv::wait`, `join`, blocking syscalls) in Dispatch/RTC.

- **CPPRT-237** MUST NOT introduce actor mailboxes, deferred event queues, or “post later” mechanisms for actor events.

- **CPPRT-238** MUST NOT write unbounded loops in Dispatch/RTC (loops MUST have a proven upper bound).

- **CPPRT-239** MUST NOT use recursion in Dispatch/RTC unless the maximum depth is a small constant proven by construction.

- **CPPRT-240** MUST NOT perform file/network/console I/O in Dispatch/RTC.

- **CPPRT-241** MUST NOT rely on dynamic initialization (static local init guards, global init order) in Dispatch/RTC.

- **CPPRT-242** MUST NOT throw exceptions in Dispatch/RTC; MUST NOT use try/catch as normal control flow there.

- **CPPRT-243** MUST NOT use `volatile` for synchronization between threads.

- **CPPRT-244** MUST NOT type-pun by `reinterpret_cast`ing pointers and dereferencing (strict aliasing UB).

- **CPPRT-245** MUST NOT rely on union type punning for portable bit reinterpretation.

- **CPPRT-246** MUST NOT use `std::function` in Dispatch/RTC hot paths.

- **CPPRT-247** MUST NOT use `std::regex` in RT-critical components (unpredictable performance and allocation).

- **CPPRT-248** MUST NOT use `std::shared_ptr` in Dispatch/RTC.

- **CPPRT-249** MUST NOT use node-based containers (`std::map`, `std::list`, `std::set`) in Dispatch/RTC hot paths.

- **CPPRT-250** MUST NOT use hash tables in Dispatch/RTC unless rehashing is impossible and allocation is proven absent.

- **CPPRT-251** MUST NOT do printf-style formatting in Dispatch/RTC hot loops.

- **CPPRT-252** MUST NOT use logging implementations that acquire locks or block in Dispatch/RTC.

- **CPPRT-253** MUST NOT use wall-clock time (`CLOCK_REALTIME`/`system_clock`) for elapsed-time measurement.

- **CPPRT-254** MUST NOT use `rand()` or global RNG state in Dispatch/RTC; randomness MUST be injected via events/config.

- **CPPRT-255** MUST NOT rely on signed integer overflow wrapping.

- **CPPRT-256** MUST NOT mix atomic and non-atomic access to the same variable across threads.

- **CPPRT-257** MUST NOT tolerate data races as “benign”; they are UB.

- **CPPRT-258** MUST NOT create/destroy threads in Dispatch/RTC.

- **CPPRT-259** MUST NOT `dlopen`/load plugins during Dispatch/RTC.


## Additional Merged Rules

- **CPPRT-001** MUST comply with the **RTC**, **no-queue**, **determinism**, **single-writer**, **no-heap-during-dispatch**, and **bounded-work** invariants defined in `sml.rules.md` for any stateforward.SML-based actor/state-machine dispatch chain.
- `stateforward::sml::sm<...>::process_event(...)` execution, including guards, actions, entry/exit actions, and anonymous/internal transitions.
- any synchronous cross-actor call that occurs inside the above chain.
- **CPPRT-003** MUST separate execution into **initialization/configuration phase** (not time critical) and **dispatch/RTC phase** (time critical).
- **CPPRT-005** MUST NOT read time, randomness, filesystem state, network state, environment variables, or global mutable process state directly from the dispatch/RTC phase.
- **CPPRT-007** MUST treat any operation with unbounded or input-dependent worst-case latency as forbidden in dispatch/RTC (including blocking I/O, paging, locks, and dynamic allocation).
- **CPPRT-008** MUST NOT create background worker threads, task pools, or asynchronous jobs from within an actor’s dispatch/RTC execution (aligns with “no message queue” and RTC invariants).
- **CPPRT-009** MUST use the terms **actor**, **dispatch**, **event**, **RTC chain**, and **dispatch-critical** consistently across the codebase and in reviews.
- **CPPRT-010** if a rule is violated for a specific subsystem, the violation MUST be:
- **CPPRT-011** MUST compile all C code as **c17** or newer (`-std=c17` or `-std=c23`).
- c11/c17 atomics (`<stdatomic.h>`) for C targets that use concurrency, or an explicit platform atomic layer.
- **CPPRT-014** MUST compile with warnings enabled and treated as errors for production builds (`-werror` or equivalent).
- **CPPRT-015** SHOULD enable at least: `-wall -wextra -wpedantic` (or MSVC equivalents) on all builds.
- **CPPRT-018** MUST NOT use `-ofast` or `-ffast-math` in dispatch/RTC binaries that require deterministic numeric behavior, because they may change IEEE/ISO semantics.
- **CPPRT-023** MUST run a CI configuration that executes tests with UB detection enabled (e.g., clang/GCC UBSan).
- **CPPRT-027** MUST obey strict aliasing rules; code MUST be correct under `-o2` with strict aliasing enabled.
- **CPPRT-033** if storage is reused via placement new for a different object lifetime, MUST use `std::launder` as required by the standard when reusing prior pointers.
- **CPPRT-037** when accessing object representations, MUST use `unsigned char`/`std::byte` pointers or `memcpy`, not incompatible typed pointers.
- **CPPRT-040** on hosted OS targets with virtual memory, MUST prevent page faults during dispatch/RTC by:
- and pre-faulting stack/working set during initialization.
- **CPPRT-041** MUST set and enforce a per-thread stack budget for dispatch/RTC threads (via linker script, RTOS config, or OS thread attributes).
- **CPPRT-043** any global/static object reachable from dispatch/RTC MUST be **constant-initialized** (no dynamic initialization).
- **CPPRT-044** MUST NOT use function-local `static` variables that require runtime initialization or guard checks inside dispatch/RTC paths (may introduce hidden locks and nondeterministic first-use latency).
- **CPPRT-001** MUST comply with the **RTC**, **no-queue**, **determinism**, **single-writer**, **no-heap-during-dispatch**, and **bounded-work** invariants defined in `sml.rules.md` for any Boost.SML-based actor/state-machine dispatch chain.
- CPPRT-001: MUST comply with the **RTC**, **no-queue**, **determinism**, **single-writer**, **no-heap-during-dispatch**, and **bounded-work** invariants defined in `sml.rules.md` for any actor or state-machine dispatch chain that follows this model.
- CPPRT-002: MUST treat the following as **dispatch-critical** (real-time constrained) code paths: `boost::sml::sm<...>::process_event(...)` execution, including guards, actions, entry/exit actions, and anonymous/internal transitions. Any synchronous cross-actor call that occurs inside the above chain.
- CPPRT-003: MUST separate execution into **Initialization/Configuration phase** (not time critical) and **Dispatch/RTC phase** (time critical). MUST document (in code comments or module docs) which functions are allowed in the Dispatch/RTC phase.
- CPPRT-004: MUST define “deterministic” as: for a fixed build (compiler + flags), fixed hardware/OS configuration, and identical initial state and input event sequence (including payloads), the observed state transitions and side effects are identical.
- CPPRT-005: MUST NOT read time, randomness, filesystem state, network state, environment variables, or global mutable process state directly from the Dispatch/RTC phase. MUST inject such information **only** via explicit event payloads or precomputed immutable configuration captured during Initialization.
- CPPRT-006: MUST assume **hard real-time constraints** for rules that affect determinism and bounded latency (unless a rule explicitly says MAY/SHOULD for soft real-time).
- CPPRT-007: MUST treat any operation with unbounded or input-dependent worst-case latency as forbidden in Dispatch/RTC (including blocking I/O, paging, locks, and dynamic allocation).
- CPPRT-008: MUST NOT create background worker threads, task pools, or asynchronous jobs from within an actor’s Dispatch/RTC execution (aligns with “no message queue” and RTC invariants).
- CPPRT-009: MUST use the terms **actor**, **dispatch**, **event**, **RTC chain**, and **dispatch-critical** consistently across the project and in reviews.
- CPPRT-010: If a rule is violated for a specific subsystem, the violation MUST be: explicitly documented near the code, covered by a dedicated test that bounds the risk, and tracked by an issue with an owner and removal plan.
- CPPRT-011: MUST compile all C code as **C17** or newer (`-std=c17` or `-std=c23`). MUST compile all C++ code as **C++20** or newer (`-std=c++20` or `-std=c++23`). MUST NOT rely on compiler extensions unless wrapped behind a portability layer and feature-tested.
- CPPRT-012: MUST declare whether each target is **freestanding** or **hosted** (C/C++ standard terms) and MUST gate library usage accordingly (e.g., no `<iostream>` in freestanding builds).
- CPPRT-013: MUST require a toolchain that correctly implements: C11/C17 atomics (`<stdatomic.h>`) for C targets that use concurrency, or an explicit platform atomic layer. C++11 atomics (`<atomic>`) for all C++ targets that use concurrency.
- CPPRT-014: MUST compile with warnings enabled and treated as errors for production builds (`-Werror` or equivalent). MUST explicitly document any warning suppressions and keep them narrowly scoped. (Ref: GCC “-pedantic / -pedantic-errors” behavior: https://gcc.gnu.org/onlinedocs/gcc/Warnings-and-Errors.html)
- CPPRT-015: SHOULD enable at least: `-Wall -Wextra -Wpedantic` (or MSVC equivalents) on all builds. SHOULD additionally enable hardening warnings appropriate for the project (e.g., `-Wconversion`, `-Wshadow`, `-Wdouble-promotion`) and fix violations rather than suppressing them.
- CPPRT-016: MUST pin and version the compiler and standard library used in CI for each target triple. MUST record compiler version and key flags in build artifacts (e.g., `--version` output embedded into `--build-info`).
- CPPRT-017: MUST define at least two build profiles:
- rt-debug: (instrumentation, sanitizers allowed, deterministic flags preserved)
- rt-release: (optimized, still deterministic, no sanitizer overhead)
- CPPRT-018: MUST NOT use `-Ofast` or `-ffast-math` in Dispatch/RTC binaries that require deterministic numeric behavior, because they may change IEEE/ISO semantics. (Ref: GCC Optimize Options `-ffast-math`: https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html)
- CPPRT-019: SHOULD use link-time dead stripping for embedded/code-size-constrained targets (`-ffunction-sections -fdata-sections` + linker GC) when supported. MUST verify that dead stripping does not remove required registration/entry points (no reliance on static initialization side effects).
- CPPRT-020: MUST use compile-time feature detection (e.g., `__has_cpp_attribute`, `__cpp_lib_*`) for optional C++23 features (such as `<expected>`), and provide fallbacks.
- CPPRT-021: SHOULD maintain a baseline flag set similar to:
- CPPRT-022: MUST treat **undefined behavior (UB)** as a release-blocking defect in all targets, including embedded and “performance-only” builds.
- CPPRT-023: MUST run a CI configuration that executes tests with UB detection enabled (e.g., Clang/GCC UBSan). (Ref: UBSan overview: https://www.chromium.org/developers/testing/undefinedbehaviorsanitizer/ )
- CPPRT-024: MUST NOT allow signed integer overflow in either C or C++ (it is UB). MUST use checked arithmetic, saturation arithmetic, or unsigned/explicitly widened arithmetic when overflow is possible. (Ref: SEI CERT C INT32-C: https://wiki.sei.cmu.edu/confluence/pages/viewpage.action?pageId=87152052)
- CPPRT-025: MUST guard against UB in shifts and bit operations: shift count MUST be in `[0, bit_width-1]` left-shift on signed values that overflows MUST NOT occur negative shift counts MUST NOT occur
- CPPRT-026: MUST NOT write expressions whose correctness depends on unspecified/undefined order of evaluation or side effects (common in C/C++). (Ref: SEI CERT C EXP30-C “Do not depend on the order of evaluation for side effects”: https://abougouffa.github.io/awesome-coding-standards/sei-cert-c-2016.pdf)
- CPPRT-027: MUST obey strict aliasing rules; code MUST be correct under `-O2` with strict aliasing enabled. MUST NOT “fix” aliasing violations by globally disabling aliasing (`-fno-strict-aliasing`) except as a temporary containment measure with a tracked bug. (Ref: GCC on strict-aliasing type punning warning: https://gcc.gnu.org/gcc-4.4/porting_to.html)
- CPPRT-028: MUST NOT type-pun by `reinterpret_cast`/pointer-cast and dereference. MUST use `std::bit_cast` (C++20+) or `memcpy` for bit-level reinterpretation. (Ref: `std::bit_cast`: https://en.cppreference.com/w/cpp/numeric/bit_cast.html)
- CPPRT-029: SHOULD use this pattern for bit reinterpretation:
- CPPRT-030: MUST NOT use `volatile` for inter-thread synchronization or atomicity. MUST use atomics or platform synchronization primitives instead. (Ref: C++ Core Guidelines CP.8: https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#cp8-dont-try-to-use-volatile-for-synchronization ; C volatile note: https://en.cppreference.com/w/c/language/volatile.html)
- CPPRT-031: MAY use `volatile` only for: memory-mapped I/O registers, signal/ISR shared flags when paired with the platform’s required barriers, and compiler-visibility constraints for special memory. MUST document the hardware contract at the declaration site.
- CPPRT-032: MUST NOT access objects outside their lifetime (including after placement-new reuse) without following the C++ object lifetime rules. (Ref: lifetime rules: https://en.cppreference.com/w/cpp/language/lifetime.html)
- CPPRT-033: If storage is reused via placement new for a different object lifetime, MUST use `std::launder` as required by the standard when reusing prior pointers. (Ref: `std::launder`: https://en.cppreference.com/w/cpp/utility/launder.html)
- CPPRT-034: MUST treat data races as UB in C/C++ and as release-blocking defects. MUST ensure that any shared writable state is protected by the single-writer invariant or by correctly ordered atomics.
- CPPRT-035: MUST NOT serialize/deserialize by `reinterpret_cast`ing structs or relying on padding/endianness. MUST use explicit serialization routines that define byte order and field widths.
- CPPRT-036: MUST restrict `reinterpret_cast` (C++) and dangerous pointer casts (C/C++) to: low-level boundary code (HAL, serialization, SIMD intrinsics wrappers), with documented justification, and with tests/sanitizer coverage. MUST NOT use casts to bypass the type system in general application logic.
- CPPRT-037: When accessing object representations, MUST use `unsigned char`/`std::byte` pointers or `memcpy`, not incompatible typed pointers.
- CPPRT-038: MUST NOT read uninitialized variables, padding, or storage. MUST initialize all fields explicitly, especially in structs that cross module boundaries.
- CPPRT-039: MUST NOT perform pointer arithmetic that produces a pointer outside the bounds of the same object (except one-past-the-end as permitted). MUST avoid “pointer provenance” violations by using indices/offsets and bounds checks.
- CPPRT-040: On hosted OS targets with virtual memory, MUST prevent page faults during Dispatch/RTC by: locking memory where appropriate (`mlockall`/equivalent), and pre-faulting stack/working set during Initialization. (Ref: `mlockall` note about reserving locked stack pages: https://man7.org/linux/man-pages/man2/mlock.2.html)
- CPPRT-041: MUST set and enforce a per-thread stack budget for Dispatch/RTC threads (via linker script, RTOS config, or OS thread attributes). MUST measure worst-case stack usage for the dispatch-critical call chain and keep a safety margin.
- CPPRT-042: MUST store dispatch-critical working state in: actor-owned structs (typically embedded in the actor object), stack allocations with statically known size, or static storage with immutable or single-writer discipline. MUST NOT allocate from the heap in dispatch-critical code.
- CPPRT-043: Any global/static object reachable from Dispatch/RTC MUST be **constant-initialized** (no dynamic initialization). MUST NOT rely on C++ dynamic initialization order across translation units.
- CPPRT-044: MUST NOT use function-local `static` variables that require runtime initialization or guard checks inside Dispatch/RTC paths (may introduce hidden locks and nondeterministic first-use latency).
- CPPRT-045: MUST explicitly align hot data structures (e.g., ring buffers, tensor tiles, atomic flags) to at least cache-line boundaries when false sharing or unaligned access can affect latency (`alignas(64)` or `std::hardware_destructive_interference_size` where supported). (Ref: `std::hardware_destructive_interference_size`: https://en.cppreference.com/w/cpp/thread/hardware_destructive_interference_size.html)
- CPPRT-046: MUST ensure that frequently written variables from different threads do not share cache lines (padding or per-thread sharding). MUST review shared structs for false sharing risks as part of performance review.
- CPPRT-047: MUST avoid accidental copies of large structs in Dispatch/RTC (pass by reference or `std::span`). MUST mark move/copy operations `=delete` or `noexcept` as appropriate to prevent unexpected copies.
- CPPRT-048: MUST document the ownership model for every long-lived buffer: owner (who allocates), lifetime (when freed), and access discipline (single-writer / read-only / atomic handoff).
- CPPRT-049: Any pointer/reference stored in actor state and used across events MUST refer to storage with a stable address across the entire lifetime of that reference. If stability cannot be guaranteed (e.g., `std::vector` growth), MUST store indices/handles instead.
- CPPRT-050: MUST actively enforce “no heap during dispatch” at runtime in debug/CI builds by trapping or counting allocations from `operator new`, `malloc`, and allocator backends while Dispatch/RTC is active.
- CPPRT-051: For C++ Dispatch/RTC binaries, MUST define a project-wide policy for `operator new/delete`: either globally disabled (`-fno-exceptions` + abort-on-new failure), or redirected to a bounded allocator whose failure mode is deterministic. MUST NOT use the default system allocator from Dispatch/RTC.
- CPPRT-052: SHOULD implement an allocation trap like:
- CPPRT-053: MUST NOT call `malloc/calloc/realloc/free` or `new/delete` in Dispatch/RTC paths (guards/actions/entry/exit). MUST treat violations as correctness bugs, not “performance issues”.
- CPPRT-054: If using arenas, MUST ensure arena capacity is fixed and sufficient for worst-case use. MUST define deterministic behavior on exhaustion (fail-fast, drop, or pre-agreed degradation) and MUST test it.
- CPPRT-055: If using `std::pmr::monotonic_buffer_resource`, MUST ensure the upstream resource cannot fall back to heap allocation during Dispatch/RTC (size buffers accordingly and/or use a non-allocating upstream). (Ref: `monotonic_buffer_resource` can allocate from upstream when buffer is exhausted: https://en.cppreference.com/w/cpp/memory/monotonic_buffer_resource.html)
- CPPRT-056: MAY use placement new only into explicitly owned storage (arena blocks, static buffers, or actor-owned raw storage). MUST pair with explicit destruction when required and MUST obey lifetime rules (`std::launder` where applicable).
- CPPRT-057: MUST use fixed-capacity containers in Dispatch/RTC, or containers whose capacity is finalized during Initialization. For `std::vector`, MUST call `reserve(max)` during Initialization and MUST NOT perform operations that can increase capacity during dispatch. (Ref: vector reallocation invalidation: https://en.cppreference.com/w/cpp/container/vector/reserve.html)
- CPPRT-058: MUST NOT build/append dynamic strings in Dispatch/RTC. MAY use `std::string_view`/`std::span<const char>` to refer to pre-existing stable storage, or use fixed-capacity string buffers.
- CPPRT-059: MUST NOT use `std::function` in Dispatch/RTC hot paths (it may allocate and adds type-erasure overhead). MAY use templates, function pointers, or a fixed-buffer function wrapper with a compile-time size limit. (Ref: `std::function` stores a callable and may allocate; small buffer optimization is not guaranteed: https://en.cppreference.com/w/cpp/utility/functional/function.html)
- CPPRT-060: MUST NOT use `<iostream>` in Dispatch/RTC (locale, formatting, and synchronization can allocate and/or lock). MAY use minimal, bounded formatting into preallocated buffers outside dispatch.
- CPPRT-061: If exceptions are enabled in any build, MUST ensure Dispatch/RTC code cannot throw and cannot allocate exception objects. SHOULD compile production real-time binaries with exceptions disabled (see Exception Policy).
- CPPRT-062: Any API that can allocate MUST either: accept an explicit allocator/arena handle, or be restricted to Initialization-only usage. MUST NOT hide allocations behind default parameters or global allocators.
- CPPRT-063: SHOULD avoid long-lived heap allocations with many size classes on embedded/edge targets. If heap is used in Initialization, SHOULD prefer arenas/pools to reduce fragmentation.
- CPPRT-064: MUST represent ownership explicitly: C++: `std::unique_ptr` (or custom intrusive owner) for owning pointers. C: explicit “create/destroy” API pairs or region allocation ownership. MUST NOT use owning raw pointers in new code.
- CPPRT-065: MUST NOT use `std::shared_ptr` in Dispatch/RTC (atomic refcounting and potential control-block allocations can add jitter). MAY use shared ownership outside dispatch when lifecycle requires it.
- CPPRT-066: Any borrowed pointer/reference used across events MUST point to storage whose lifetime exceeds the actor lifetime or is otherwise proven stable. MUST NOT store references to stack objects beyond the call where they are created.
- CPPRT-067: MUST NOT return or store `std::string_view`/`std::span` pointing into temporary objects or containers that may reallocate. MUST ensure the referenced storage outlives the view.
- CPPRT-068: In Dispatch/RTC, MUST avoid moves that can be linear-time (e.g., moving large vectors when allocator propagation triggers copies). MUST prefer fixed-size buffers or handles.
- CPPRT-069: Event payload types used in SML dispatch SHOULD be trivially copyable and trivially destructible. MUST NOT embed ownership or dynamic containers in event payloads unless proven allocation-free and bounded.
- CPPRT-070: Resources with expensive teardown (GPU handles, file descriptors, sockets) MUST be acquired and released outside Dispatch/RTC, and MUST be referenced by stable handles inside Dispatch/RTC.
- CPPRT-071: MUST NOT use union type-punning for portability across compilers/ABIs. MUST use `memcpy`/`std::bit_cast` and explicit endianness conversion.
- CPPRT-072: If pointer tagging is used (common in ML runtimes), it MUST be: restricted to clearly documented bit patterns, validated with static assertions on alignment, and confined to a portability layer.
- CPPRT-073: MUST ensure destructor work is bounded and deterministic. MUST NOT perform blocking I/O, locks, or heap frees in destructors that can run in Dispatch/RTC context.
- CPPRT-074: MUST enforce the **single-writer invariant**: during any RTC chain, at most one thread executes inside a given actor’s state machine (`process_event`). MUST centralize dispatch through an orchestrator that enforces this (thread affinity or external serialization).
- CPPRT-075: MUST NOT re-enter the same actor’s `process_event` recursively or via callbacks during its own dispatch (violates RTC bounded-work and complicates determinism).
- CPPRT-076: MUST NOT use: `std::async`, `std::future`, `std::promise`, thread pools, coroutines that suspend/resume asynchronously, in Dispatch/RTC paths.
- CPPRT-077: MUST create threads only during Initialization (or an explicit lifecycle stage) and MUST NOT create or destroy threads from Dispatch/RTC.
- CPPRT-078: MUST treat the following as **not Dispatch/RTC safe** unless proven otherwise: heap allocators (internal locks), locale/timezone APIs, environment access (`getenv`), iostreams, `std::regex`, dynamic loader APIs.
- CPPRT-079: MUST NOT call blocking waits (`mutex::lock`, `condition_variable::wait`, `join`, `sleep`, blocking syscalls) in Dispatch/RTC.
- CPPRT-080: Any cross-thread communication MUST: be explicitly documented as a boundary, be bounded (no unbounded buffering), preserve determinism of actor state transitions.
- CPPRT-081: MUST NOT implement actor mailboxes, deferred queues, or “post/emit later” systems for actor events. Asynchronous inputs MUST be converted into synchronous events by the orchestrator using a deterministic policy (drop/overwrite/backpressure), not by queueing.
- CPPRT-082: SHOULD pin Dispatch/RTC threads to cores (or RTOS priorities) when jitter matters. MUST document the scheduling model (one actor per thread, actor groups per thread, etc.) and keep it stable.
- CPPRT-083: SHOULD avoid thread-local storage reads/writes in Dispatch/RTC hot paths (can be cheap but may hinder portability and toolability). MUST NOT use TLS with dynamic initialization in Dispatch/RTC.
- CPPRT-084: SHOULD use concurrency analyzers where applicable (TSan in CI for hosted targets; Clang Thread Safety Analysis annotations if locks are used). (Ref: Clang Thread Safety Analysis: https://clang.llvm.org/docs/ThreadSafetyAnalysis.html)
- CPPRT-085: If a hosted OS real-time scheduler is used, MUST design to avoid priority inversion (prefer no locks; otherwise use priority inheritance/protection as required by the platform). (Ref: POSIX priority inheritance: https://pubs.opengroup.org/onlinepubs/7908799/xsh/pthread_mutexattr_setprotocol.html)
- CPPRT-086: MUST treat locks as forbidden in Dispatch/RTC by default (mutexes, RW locks, semaphores, futexes, OS critical sections).
- CPPRT-087: If locks are required, they MUST be confined to explicit boundary modules (I/O, OS integration) and MUST NOT be used inside SML guards/actions/entry/exit.
- CPPRT-088: If a lock must be used in any time-critical thread, MUST use a non-blocking acquisition strategy (`try_lock` with bounded retries) or a design that eliminates contention. MUST define and test the worst-case bound.
- CPPRT-089: On platforms with priority scheduling, any unavoidable mutex in a real-time thread MUST use priority inheritance or priority ceiling/protection where supported. (Ref: priority inheritance semantics: https://man7.org/linux/man-pages/man3/pthread_mutexattr_getprotocol.3p.html)
- CPPRT-090: If more than one lock exists in the system, MUST define a global lock order and MUST enforce it (static analysis or code review checklists). MUST NOT acquire locks out of order.
- CPPRT-091: Any allocator reachable from Dispatch/RTC MUST be lock-free or single-thread confined by construction. MUST NOT call the system allocator from Dispatch/RTC because it typically uses internal locks.
- CPPRT-092: Logging/tracing performed in Dispatch/RTC MUST NOT acquire locks; it MUST be non-blocking and bounded (see Logging rules).
- CPPRT-093: MUST NOT use lock-based memory reclamation patterns in Dispatch/RTC hot paths (e.g., global freelists with mutexes). MUST use per-actor or per-thread pools, or non-blocking designs with bounded work.
- CPPRT-094: MUST use atomics only where data crosses threads/cores/ISRs. Within a single-writer actor, MUST prefer plain non-atomic loads/stores.
- CPPRT-095: In C++ code, MUST explicitly specify `std::memory_order` for every atomic `load/store/exchange/compare_exchange` in real-time components. MUST NOT rely on default `seq_cst` implicitly. (Ref: `std::memory_order`: https://en.cppreference.com/w/cpp/atomic/memory_order.html)
- CPPRT-096: MAY use `memory_order_relaxed` for independent counters/statistics that do not publish data and do not participate in synchronization.
- CPPRT-097: MUST use **release** on the publishing side and **acquire** on the consuming side for ownership/data handoff patterns. MUST document the handoff variable and the data it protects.
- CPPRT-098: SHOULD avoid `memory_order_seq_cst` in hot paths unless a global total order is required and justified; it can add fences and jitter.
- CPPRT-099: MUST NOT use `atomic_thread_fence` unless implementing a documented low-level pattern. Any fence usage MUST include a comment that states the synchronization contract. (Ref: `std::atomic_thread_fence`: https://en.cppreference.com/w/cpp/atomic/atomic_thread_fence.html)
- CPPRT-100: MUST NOT use atomic operations on bit-fields or rely on bit-field layout across compilers; use full-width atomics instead.
- CPPRT-101: MUST align and pad atomics that are written frequently or by different threads to avoid false sharing. SHOULD co-locate read-mostly atomics separately from write-heavy atomics.
- CPPRT-102: MUST NOT access the same memory location sometimes atomically and sometimes non-atomically across threads (data race/UB). MUST encapsulate the variable behind a single access API.
- CPPRT-103: MUST handle spurious failures correctly (`compare_exchange_weak` in a loop). MUST use `compare_exchange_strong` only when necessary.
- CPPRT-104: If using C++20 atomic wait/notify, MUST confine it to boundary threads (not actor Dispatch/RTC) and MUST bound waiting time.
- CPPRT-105: If an ISR writes data read by a Dispatch/RTC thread, MUST use: atomic flags for “data ready” publication, and a deterministic overwrite/drop policy (no unbounded buffering). MUST NOT perform dynamic allocation or locking in the ISR.
- CPPRT-106: MUST compile real-time/dispatch-critical C++ components with exceptions **disabled** (`-fno-exceptions` or equivalent) unless an explicit exception is granted for a boundary-only module.
- CPPRT-107: All functions that may execute during Dispatch/RTC (guards/actions/entry/exit and anything they call) MUST be `noexcept` (directly or effectively). MUST treat any potential throw path as a defect.
- CPPRT-108: MUST NOT use exceptions for control flow, retries, or expected failure in any part of the system.
- CPPRT-109: If any linked code can throw (third-party libs, non-RT subsystems), MUST define an explicit boundary wrapper that: catches all exceptions, converts to a deterministic error signal, and prevents exceptions from propagating into actor Dispatch/RTC.
- CPPRT-110: MUST ensure destructors are `noexcept` (the default in modern C++), and MUST NOT throw from destructors.
- CPPRT-111: SHOULD note in project docs that exceptions can be unacceptable in life-critical hard-real-time code due to control-flow and unwind-cost unpredictability. (Ref: C++ Core Guidelines resource management note: https://cpp-core-guidelines-docs.vercel.app/resource)
- CPPRT-112: MUST represent recoverable/expected errors without exceptions: C: return error codes or status structs. C++: return status objects (e.g., `std::expected<T,E>` in C++23) or error codes. (Ref: `std::expected` (C++23): https://en.cppreference.com/w/cpp/utility/expected.html)
- CPPRT-113: Error/status return types used in Dispatch/RTC MUST be trivially copyable/movable and MUST NOT allocate. MUST avoid storing strings or dynamic data in error objects in Dispatch/RTC.
- CPPRT-114: MUST mark fallible functions as `[[nodiscard]]` (C++), or enforce via static analysis in C, so errors are not silently ignored.
- CPPRT-115: MUST propagate errors explicitly through return values, not via global state or hidden thread-local state. MUST keep error propagation bounded (no retries with unbounded loops in Dispatch/RTC).
- CPPRT-116: MUST classify failures into:
- recoverable: (modeled as state transitions / error returns),
- fatal: (leads to deterministic fail-stop or fault state),
- external: (I/O/service down; handled outside dispatch). MUST document classification for each module boundary.
- CPPRT-117: On fatal invariant violation, MUST take a deterministic action: enter a defined “fault” state and stop processing further events, OR terminate the process with a clear diagnostic. MUST NOT attempt best-effort recovery in Dispatch/RTC after invariant failure.
- CPPRT-118: MUST NOT rely on `errno` in Dispatch/RTC code. If interacting with syscalls, MUST capture error codes at the boundary and convert to stable domain errors before dispatch.
- CPPRT-119: MUST ensure error creation/propagation does not implicitly log or allocate in Dispatch/RTC. Logging on error MUST follow bounded logging rules.
- CPPRT-120: In C++, MAY use `[[noreturn]]` and `std::terminate`/`abort` for fatal paths. MUST ensure any `noreturn` path does not perform unbounded work.
- CPPRT-121: MUST standardize the project’s error conventions (success code, error enum ranges, and mapping to OS errors). MUST NOT create ad-hoc error code meanings per module.
- CPPRT-122: MUST clearly label APIs as one of:
- RT-safe: (may be called in Dispatch/RTC),
- Init-only: (must not be called in Dispatch/RTC),
- Boundary: (I/O/OS integration; called outside Dispatch/RTC). This labeling MUST be visible in headers (comments, attributes, or naming conventions).
- CPPRT-123: Any API that may allocate MUST state so in its documentation and MUST NOT be callable from RT-safe contexts. RT-safe APIs MUST be allocation-free by construction.
- CPPRT-124: RT-safe APIs MUST NOT block, wait, sleep, or perform syscalls with unpredictable latency.
- CPPRT-125: RT-safe APIs SHOULD accept `std::span<T>`/`std::span<const T>` (C++20) or `(ptr,len)` pairs (C) for buffer parameters. MUST NOT accept owning containers as inputs in RT-safe APIs. (Ref: `std::span`: https://en.cppreference.com/w/cpp/container/span.html)
- CPPRT-126: MUST make sizes, counts, and time units explicit in API types (e.g., `std::chrono::nanoseconds`, `size_t`). MUST NOT pass “raw ints” with ambiguous units.
- CPPRT-127: APIs MUST not expose partially initialized objects. Constructors/factories MUST return fully initialized objects or an error status (no half-valid instances).
- CPPRT-128: Actor boundary APIs (event ingress/egress) MUST use stable POD-like event types and MUST avoid ABI-unstable STL types across shared-library boundaries.
- CPPRT-129: MUST NOT accept callbacks in RT-safe APIs that can re-enter actor dispatch or call back into unknown code during Dispatch/RTC (breaks bounded-work reasoning).
- CPPRT-130: MUST model long-running operations as explicit state transitions driven by events, not as blocking calls or internal background work.
- CPPRT-131: Public APIs MUST include a deprecation mechanism (compile-time attribute or macro) and a removal policy. MUST NOT silently change semantics of RT-safe APIs without versioning.
- CPPRT-132: MUST NOT expose C++ standard library types (e.g., `std::string`, `std::vector`, `std::expected`) in stable ABI boundaries between separately compiled/shared components. MUST use C ABI (`extern "C"`) and POD structs for stable interfaces.
- CPPRT-133: SHOULD compile libraries with hidden symbol visibility by default and explicitly export the public API surface (`-fvisibility=hidden` + export macros on ELF platforms).
- CPPRT-134: Any struct used in binary interfaces MUST be: `std::is_standard_layout_v == true`, `std::is_trivially_copyable_v == true`, and validated with `static_assert` on size and offsets where relevant.
- CPPRT-135: MUST define endianness for every serialized/binary format. MUST use explicit byte-order conversions; MUST NOT assume host endianness.
- CPPRT-136: MUST NOT use `#pragma pack`/`__attribute__((packed))` for performance-critical data that is frequently accessed; it can cause misaligned loads and UB on some targets. MAY use packed structs only for wire parsing, followed immediately by explicit decoding into aligned native structs.
- CPPRT-137: MUST `static_assert(alignof(T) >= N)` when code relies on alignment (SIMD loads, pointer tagging). MUST provide fallback paths when alignment cannot be guaranteed.
- CPPRT-138: MUST avoid One Definition Rule violations across translation units and packages (common in header-only template code). MUST centralize configuration macros in a single header and keep them consistent.
- CPPRT-139: If providing a C ABI, MUST use stable integer/enum error codes and MUST document them as part of the ABI contract.
- CPPRT-140: MUST initialize padding bytes in structs that cross trust boundaries or are hashed/serialized, to avoid information leaks and nondeterministic comparisons.
- CPPRT-141: For libraries with stable ABI, MUST maintain ABI compatibility tests (e.g., size/layout checks and symbol checks) across releases.
- CPPRT-142: MUST prefer contiguous storage (`std::array`, `std::span`, fixed-capacity vectors, SoA layouts) for hot-path data. MUST justify pointer-chasing structures in performance-critical code.
- CPPRT-143: MUST NOT use `std::list`, intrusive lists, or general linked lists in Dispatch/RTC hot paths unless a benchmark proves it is superior and work is bounded.
- CPPRT-144: If `std::vector` is used in RT-safe code, its capacity MUST be fixed during Initialization (`reserve`) and MUST NOT grow in Dispatch/RTC. MUST treat any reallocation as a correctness violation.
- CPPRT-145: MUST NOT use `std::unordered_map`/`std::unordered_set` in Dispatch/RTC unless: the allocator is fixed/pool-based, rehashing is impossible (reserved and load-factor bounded), and worst-case probe bounds are acceptable and tested.
- CPPRT-146: SHOULD use sorted vectors (“flat_map” style) for small or mostly-static keyspaces to improve cache locality and predictability.
- CPPRT-147: For CPU-bound ML engines, SHOULD prefer Structure-of-Arrays (SoA) or blocked/tiled layouts for tensor metadata and hot activation buffers to improve cache utilization and SIMD friendliness.
- CPPRT-148: MUST avoid implicit buffer copies in API boundaries (return-by-value of large buffers, passing large structs by value). MUST pass by `span` or reference and keep ownership explicit.
- CPPRT-149: If ring buffers or producer/consumer buffers are used for logging or boundary telemetry, MUST place producer and consumer indices on separate cache lines to avoid false sharing. (Ref: Linux tracing ring buffer design emphasizes lockless/bounded behavior: https://docs.kernel.org/trace/ring-buffer-design.html)
- CPPRT-150: MUST avoid allocator-heavy abstractions (polymorphic allocators with upstream heap, node-based containers) in hot paths unless bounded and proven allocation-free.
- CPPRT-151: MUST define memory budgets for: actor state, per-thread scratch buffers, and per-subsystem arenas, and MUST fail deterministically if budgets are exceeded.
- CPPRT-152: MAY use explicit prefetch intrinsics only when profiling demonstrates benefit. MUST keep prefetch usage behind a portability macro and benchmark across target CPUs.
- CPPRT-153: MUST NOT perform dynamic formatting (printf-like formatting) inside hot loops; log numeric codes or sample data into preallocated buffers.
- CPPRT-154: In hot paths, SHOULD minimize unpredictable branches (data-dependent conditionals) and SHOULD prefer data layouts that improve predictability. MUST justify manual branch prediction hints (`__builtin_expect`) with profiling evidence.
- CPPRT-155: MUST NOT use virtual function calls in Dispatch/RTC hot loops where bounded latency matters. MUST justify any virtual dispatch with profiling evidence and boundedness arguments.
- CPPRT-156: MUST NOT use RTTI (`dynamic_cast`, `typeid`) in Dispatch/RTC. MAY use RTTI only in Initialization, diagnostics, or tooling builds.
- CPPRT-157: SHOULD prefer templates, CRTP, or `std::variant` visitation for polymorphism in hot paths. MUST keep variant alternatives bounded and known at compile time.
- CPPRT-158: MUST avoid general type erasure (`std::any`, `std::function`, `std::move_only_function`) in RT-critical paths unless the implementation is proven allocation-free and bounded.
- CPPRT-159: MUST NOT rely on dynamic loader plugins (dlopen/loadlibrary) or late-binding function resolution in Dispatch/RTC. All function pointers used in Dispatch/RTC MUST be resolved during Initialization.
- CPPRT-160: MUST NOT delete through a base pointer in Dispatch/RTC (dynamic allocation and virtual destructors are forbidden there). Object graphs MUST have deterministic lifetimes managed outside dispatch.
- CPPRT-161: If using `std::variant`, MUST keep visitation logic simple and MUST avoid nested variants that can cause combinatorial visitation code growth in hot paths.
- CPPRT-162: MAY use explicit vtable structs (manual function pointer tables) for ABI-stable hot-path polymorphism, but MUST ensure: tables are immutable, calls are bounded, and initialization happens before Dispatch/RTC.
- CPPRT-163: MUST prefer compile-time validation (`static_assert`, `constexpr` checks) for invariants that can be proven at compile time (sizes, ranges, alignment, enum completeness).
- CPPRT-164: Any lookup table used in Dispatch/RTC SHOULD be `constexpr` and stored in read-only memory. MUST NOT lazily initialize such tables at first use inside dispatch.
- CPPRT-165: MUST keep template recursion and compile-time computation bounded to avoid runaway compile times and code size. MUST set and enforce build time budgets in CI for large monorepos.
- CPPRT-166: MUST NOT generate code or JIT at runtime in Dispatch/RTC. If JIT/codegen is required for ML engines, it MUST occur during Initialization with deterministic inputs.
- CPPRT-167: SHOULD use explicit template instantiation in `.cc/.cpp` files for heavy templates to reduce compile times and improve link determinism.
- CPPRT-168: MUST NOT rely on constexpr evaluation that differs across compilers (non-portable intrinsics). MUST keep constexpr logic within standard-defined behavior.
- CPPRT-169: If using code generation (protobuf, flatbuffers, custom), MUST pin generator versions and MUST treat generated code as part of the reproducible build contract.
- CPPRT-170: MUST keep compile-time heavy components (templates, generated code) behind clear module boundaries to prevent cascading rebuilds.
- CPPRT-171: MUST choose optimization levels (`-O2` vs `-O3`) based on measured throughput and latency jitter on target hardware. MUST keep a documented baseline per target.
- CPPRT-172: MUST NOT use `-Ofast` or `-ffast-math` for deterministic real-time builds (can change floating-point semantics and break reproducibility). (Ref: GCC Optimize Options: https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html)
- CPPRT-173: If bitwise-deterministic floating point results are required across CPUs, MUST control FP contraction and reassociation (e.g., disable contraction where needed) and MUST document the chosen policy per target.
- CPPRT-174: MAY enable LTO for throughput, but MUST: measure its impact on latency/jitter, keep the link step deterministic (pinned toolchain), and ensure debug symbol strategy is workable for incident response.
- CPPRT-175: MAY enable PGO for throughput, but MUST: collect profiles on representative workloads, treat profile data as a versioned build input, and measure worst-case latency effects (branch layout changes can affect i-cache behavior).
- CPPRT-176: MUST avoid `always_inline` abuse. MAY use `[[gnu::always_inline]]`/`__forceinline` only when profiling proves benefit and code size impact is acceptable.
- CPPRT-177: SHOULD keep frame pointers in at least one shipping profile for production observability if it does not violate latency budgets (`-fno-omit-frame-pointer` on many targets).
- CPPRT-178: MUST NOT rely on flags that change language semantics to “hide” UB (e.g., `-fwrapv`) as a general policy. If used for legacy compatibility, MUST still treat overflow/UB as defects and plan removal.
- CPPRT-179: On hosted ELF targets, SHOULD enable standard linker hardening flags (RELRO, NOW, PIE) unless they conflict with real-time constraints; any disablement MUST be documented.
- CPPRT-180: MUST ensure that build outputs do not depend on: absolute paths in debug info (unless stripped), timestamps (use reproducible build flags where available), or non-versioned generated sources.
- CPPRT-181: Logging/tracing callable from Dispatch/RTC MUST be: non-blocking, allocation-free, and bounded O(1) per call.
- CPPRT-182: MUST NOT perform syscalls or device I/O (write to files, sockets, stdout) from Dispatch/RTC logging. MUST buffer logs into preallocated memory and flush outside Dispatch/RTC.
- CPPRT-183: SHOULD implement RT-safe logging as a fixed-size ring buffer with overwrite-or-drop behavior on overflow. (Ref: Linux kernel lockless ring buffer supports overwrite/producer-consumer modes: https://docs.kernel.org/trace/ring-buffer-design.html)
- CPPRT-184: MUST define a deterministic overflow policy for log buffers (drop newest, drop oldest, overwrite oldest) and MUST test it.
- CPPRT-185: MUST NOT make functional decisions based on whether logging succeeds (e.g., “if log buffer full then change behavior”) in Dispatch/RTC. Logging is observability only.
- CPPRT-186: MUST avoid string formatting in Dispatch/RTC. SHOULD log numeric event IDs + small fixed payloads, or format later during offline decode.
- CPPRT-187: If using `std::source_location` for diagnostics, MUST ensure it does not leak absolute paths into production artifacts and MUST keep it out of Dispatch/RTC hot paths unless proven bounded. (Ref: `std::source_location`: https://en.cppreference.com/w/cpp/utility/source_location.html)
- CPPRT-188: On embedded targets, MAY use a non-blocking real-time transfer mechanism (e.g., SEGGER RTT) provided it is bounded and does not allocate. (Ref: SEGGER RTT design goal is real-time transfer without halting the CPU: https://www.segger.com/products/debug-probes/j-link/technology/about-real-time-transfer/ )
- CPPRT-189: MUST compile-time gate trace points (`#if TRACE_ENABLED`) to allow zero-overhead removal in production profiles when required.
- CPPRT-190: MUST perform any aggregation, compression, encoding, or export of telemetry outside Dispatch/RTC in a bounded, scheduled boundary stage.
- CPPRT-191: MUST NOT read wall-clock or monotonic time directly inside Dispatch/RTC. MUST obtain time in the orchestrator/boundary layer and inject it into the actor as part of an event payload when needed for decisions.
- CPPRT-192: Boundary/orchestrator code MUST use a monotonic clock for measuring durations (not `CLOCK_REALTIME`). On Linux, SHOULD use `CLOCK_MONOTONIC` or `CLOCK_MONOTONIC_RAW` depending on your NTP/adjustment requirements. (Ref: `clock_gettime` clock IDs: https://man7.org/linux/man-pages/man2/clock_gettime.2.html)
- CPPRT-193: On hosted OS targets, MUST explicitly configure scheduling policy/priority for Dispatch/RTC threads (where supported) and MUST test behavior under load. (Ref: Linux scheduling overview: https://man7.org/linux/man-pages/man7/sched.7.html)
- CPPRT-194: MUST NOT call `sleep`, `nanosleep`, `sched_yield`, or equivalent from Dispatch/RTC. Any waiting MUST be moved to the orchestrator and expressed as future events/timers.
- CPPRT-195: MUST centralize timer management in a single boundary module. Actors MUST receive timer expirations as explicit events and MUST NOT arm OS timers from Dispatch/RTC.
- CPPRT-196: If any blocking primitives exist in boundary code, MUST ensure they do not introduce priority inversion for real-time threads (priority inheritance/protection as appropriate). (Ref: POSIX mutex protocol attributes: https://pubs.opengroup.org/onlinepubs/7908799/xsh/pthread_mutexattr_setprotocol.html)
- CPPRT-197: If using memory locking to avoid page faults, MUST do so before entering the real-time section and MUST pre-touch stack/heap pages needed for Dispatch/RTC. (Ref: `mlockall` stack prefault advice: https://manpages.ubuntu.com/manpages/jammy/man2/mlock.2.html)
- CPPRT-198: MUST treat syscalls as boundary-only operations. Actors MUST not call syscalls from Dispatch/RTC (file/network/clock/scheduler APIs).
- CPPRT-199: MUST define per-event and per-actor time budgets. MUST measure and alert/abort deterministically on budget violations (e.g., watchdog event that triggers a fault state).
- CPPRT-200: MUST test timer/event behavior under CPU saturation and I/O pressure to ensure deadlines and ordering remain within spec.
- CPPRT-201: MUST perform syscalls only in boundary/orchestrator code (I/O, scheduling, memory management). MUST NOT call syscalls from within actor Dispatch/RTC.
- CPPRT-202: Boundary I/O MUST be configured as non-blocking where feasible. MUST bound per-iteration work when polling I/O (no unbounded draining loops).
- CPPRT-203: MUST NOT allocate, lock, or perform heavy work in interrupt context. ISR code MUST be minimal and MUST communicate with the orchestrator/actor via bounded shared state (latest-value + atomic flag).
- CPPRT-204: MUST isolate memory-mapped I/O access behind a dedicated HAL (hardware abstraction layer). MUST NOT scatter `volatile` register accesses throughout actor logic.
- CPPRT-205: For MMIO registers, MUST use `volatile` qualified access as required to prevent elision, and MUST apply the platform’s required memory barriers/fences for ordering with devices. MUST document the ordering requirements per device.
- CPPRT-206: MUST NOT perform misaligned loads/stores via pointer casts. MUST use `memcpy` or architecture-provided unaligned access helpers when decoding packed wire/device formats.
- CPPRT-207: MUST avoid Unix signals and signal handlers in Dispatch/RTC threads (asynchronous and hard to bound). If signals are required, MUST handle them in a dedicated boundary thread.
- CPPRT-208: MUST wrap OS/hardware APIs behind thin adapters that: translate errors to project status types, normalize time units, and make blocking/allocating behavior explicit.
- CPPRT-209: MUST bound any device-driver interaction loops (e.g., draining RX descriptors) and MUST enforce maximum work per dispatch tick.
- CPPRT-210: MUST NOT load/unload shared libraries or resolve symbols dynamically in Dispatch/RTC. If dynamic loading is required, MUST do it during Initialization only.
- CPPRT-211: For hosted targets, MUST run CI jobs with: AddressSanitizer (ASan), UndefinedBehaviorSanitizer (UBSan), and ThreadSanitizer (TSan) where concurrency exists, on representative test suites. (Ref: Clang sanitizer documentation: https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html)
- CPPRT-212: MUST NOT ship sanitizers in production real-time binaries unless explicitly justified (they change timing and may allocate). Sanitizers are for CI/testing.
- CPPRT-213: MUST gate merges on static analysis for safety-critical code: clang-tidy (bugprone/performance/readability), and at least one additional analyzer (Clang Static Analyzer, GCC -fanalyzer, or commercial tool). (Ref: clang-tidy docs: https://clang.llvm.org/extra/clang-tidy/ ; Clang Static Analyzer: https://clang.llvm.org/docs/ClangStaticAnalyzer.html)
- CPPRT-214: SHOULD align coding rules with an established secure/safety standard (CERT C/C++, MISRA C/C++ or AUTOSAR C++) and MUST document deviations. (Ref: Cppcheck MISRA addon notes: https://cppcheck.sourceforge.io/manual.html)
- CPPRT-215: MUST validate all untrusted inputs at trust boundaries (network packets, files, IPC, hardware DMA buffers). Parsing/validation MUST be separated from Dispatch/RTC logic and MUST produce validated, bounded data structures before dispatch.
- CPPRT-216: Bit-level and SIMD code MUST be reviewed for UB (alignment, aliasing, overflow, shifts). MUST include targeted tests and UBSan coverage for these modules.
- CPPRT-217: MUST perform explicit, checked conversions between integer sizes and signedness. MUST NOT rely on implicit narrowing conversions.
- CPPRT-218: On hosted targets, SHOULD enable stack protection and fortify features (`-fstack-protector-strong`, `_FORTIFY_SOURCE`) where supported and compatible with latency requirements. Any disablement MUST be documented with rationale.
- CPPRT-219: MUST NOT store credentials, private keys, or secrets in source control. MUST scan repositories for secrets in CI.
- CPPRT-220: MUST pin toolchain and third-party dependency versions (including compiler, libc++, libstdc++, and embedded SDKs) for reproducible, auditable builds.
- CPPRT-221: SHOULD fuzz parsers/decoders and boundary protocol handling (libFuzzer/AFL++/Honggfuzz) in CI for hosted targets. MUST keep fuzz targets separate from Dispatch/RTC binaries.
- CPPRT-222: MUST initialize memory before use (no uninitialized reads). MUST avoid “use of uninitialized padding” in comparisons/hashes (zero padding or use field-wise operations).
- CPPRT-223: MUST have automated tests that replay recorded event sequences and assert identical: state transitions, actor outputs, and (when enabled) trace events.
- CPPRT-224: MUST have tests that verify **zero dynamic allocation** occurs during Dispatch/RTC by: trapping `new/malloc`, or using allocator instrumentation, and failing the test on any allocation.
- CPPRT-225: MUST have tests that assert per-event upper bounds on: number of transitions/actions, loop iterations in hot paths, and maximum batch sizes processed per dispatch tick.
- CPPRT-226: SHOULD run latency/jitter microbenchmarks on representative hardware in CI or nightly jobs. MUST treat regressions beyond a defined threshold as failures requiring triage.
- CPPRT-227: If multi-threading exists at boundaries, MUST run ThreadSanitizer (hosted targets) and stress tests to detect data races and ordering bugs.
- CPPRT-228: MUST keep fuzzing/sanitizer-heavy test binaries separate from production RT binaries. MUST ensure fuzz targets cover boundary parsing/decoding logic.
- CPPRT-229: MUST run clang-tidy/cppcheck/static analyzer as part of CI and MUST treat new findings in RT-critical modules as failures.
- CPPRT-230: For embedded targets, SHOULD run hardware-in-the-loop tests that validate timing, scheduling, and device interactions under load.
- CPPRT-231: MUST include fault-injection tests for boundary failures (I/O errors, corrupted inputs, allocation exhaustion during init) and verify deterministic error handling.
- CPPRT-232: MUST define coverage expectations (statement/branch) for safety-critical code and MUST ensure coverage is measured in CI for hosted targets where feasible.
- CPPRT-233: If using golden output traces, MUST ensure they are stable across platforms/builds for the supported targets, or MUST scope them to a specific platform/toolchain.
- CPPRT-234: Performance profiling runs MUST be repeatable: pinned CPU frequency governors (hosted OS), fixed thread affinity, and controlled background load. MUST document the profiling protocol.
- CPPRT-235: MUST NOT allocate from heap (`new`, `malloc`, container growth) anywhere in Dispatch/RTC.
- CPPRT-236: MUST NOT call blocking waits (`mutex::lock`, `cv::wait`, `join`, blocking syscalls) in Dispatch/RTC.
- CPPRT-237: MUST NOT introduce actor mailboxes, deferred event queues, or “post later” mechanisms for actor events.
- CPPRT-238: MUST NOT write unbounded loops in Dispatch/RTC (loops MUST have a proven upper bound).
- CPPRT-239: MUST NOT use recursion in Dispatch/RTC unless the maximum depth is a small constant proven by construction.
- CPPRT-240: MUST NOT perform file/network/console I/O in Dispatch/RTC.
- CPPRT-241: MUST NOT rely on dynamic initialization (static local init guards, global init order) in Dispatch/RTC.
- CPPRT-242: MUST NOT throw exceptions in Dispatch/RTC; MUST NOT use try/catch as normal control flow there.
- CPPRT-243: MUST NOT use `volatile` for synchronization between threads.
- CPPRT-244: MUST NOT type-pun by `reinterpret_cast`ing pointers and dereferencing (strict aliasing UB).
- CPPRT-245: MUST NOT rely on union type punning for portable bit reinterpretation.
- CPPRT-246: MUST NOT use `std::function` in Dispatch/RTC hot paths.
- CPPRT-247: MUST NOT use `std::regex` in RT-critical components (unpredictable performance and allocation).
- CPPRT-248: MUST NOT use `std::shared_ptr` in Dispatch/RTC.
- CPPRT-249: MUST NOT use node-based containers (`std::map`, `std::list`, `std::set`) in Dispatch/RTC hot paths.
- CPPRT-250: MUST NOT use hash tables in Dispatch/RTC unless rehashing is impossible and allocation is proven absent.
- CPPRT-251: MUST NOT do printf-style formatting in Dispatch/RTC hot loops.
- CPPRT-252: MUST NOT use logging implementations that acquire locks or block in Dispatch/RTC.
- CPPRT-253: MUST NOT use wall-clock time (`CLOCK_REALTIME`/`system_clock`) for elapsed-time measurement.
- CPPRT-254: MUST NOT use `rand()` or global RNG state in Dispatch/RTC; randomness MUST be injected via events/config.
- CPPRT-255: MUST NOT rely on signed integer overflow wrapping.
- CPPRT-256: MUST NOT mix atomic and non-atomic access to the same variable across threads.
- CPPRT-257: MUST NOT tolerate data races as “benign”; they are UB.
- CPPRT-258: MUST NOT create/destroy threads in Dispatch/RTC.
- CPPRT-259: MUST NOT `dlopen`/load plugins during Dispatch/RTC.
