# golang.rules.md (2025-2026)

This is a hard policy ruleset for AI coding agents working in real-world Go projects, optimized for performance, memory efficiency, and scalable concurrency.

Baseline: Go 1.26.x (current as of Feb 2026) and its standard library capabilities (new default GC, goroutine leak profile experiment, etc.). See [S1].

## 0. Non-negotiables (always enforced)

0.1 No CGO. Build and test with `CGO_ENABLED=0`. Any PR that introduces `import "C"`, `// #cgo`, or requires a C toolchain is rejected. (We do not “make an exception” because the runtime performance and portability trade-offs leak everywhere.) See [S32].

0.2 Dependencies must be minimal.
1) Prefer standard library.
2) Allowed third-party runtime state-machine dependency: a single approved HSM library only, and only for explicitly modeled stateful behavior. See [S30].
3) Allowed third-party documentation tool dependency: a single approved README generator only, when documentation generation is required. See [S29].
4) Any other dependency requires explicit written justification in the PR description (what stdlib option failed, measurable cost/benefit, and a removal plan). Permission for such dependencies may be explicitly asked for or granted in requirements.

0.3 Testing must use only the builtin `testing` package. No external test frameworks. See [S14], [S16], [S17].

0.4 Docs live with the code. If README generation is part of the project standard, generated READMEs must match the checked-in output. See [S29].

0.5 Stateful behavior MUST use the project-approved HSM library.
Any workflow, lifecycle, protocol handler, orchestration logic, or multi-step behavior must be expressed as an HSM model plus events and operations. See [S30].

Forbidden
- Adding a new concurrency-heavy workflow using ad-hoc goroutines, callback chains, or implicit state in scattered booleans.
- “Just make it single threaded” as a performance workaround.

## 1. Toolchain and build policy

Why this rule exists: Performance and concurrency behavior can change across Go releases. Policy depends on repeatable builds and consistent semantics. See [S18], [S19].

1.1 Pin the toolchain.
- `go.mod` MUST include a `toolchain` line pinned to an exact patch version (example: `toolchain go1.26.0`) and a `go` line pinned to a Go language version used by the project (example: `go 1.26.0`). See [S19], [S20].

1.2 Upgrade policy.
- When upgrading Go, run `go fix` and commit the resulting changes as a standalone PR (no behavioral changes mixed in). See [S28].

1.3 Container CPU limits.
- Do not vendor or depend on `automaxprocs`. Go 1.25+ is container-aware by default; do not override `GOMAXPROCS` unless you have measured tail-latency impact for a specific workload. See [S4], [S2].

1.4 Experiments policy.
- `GOEXPERIMENT` MUST be empty in production builds unless the project explicitly opts in.
- Allowed in CI-only builds:
  - `GOEXPERIMENT=goroutineleakprofile` for leak detection (Go 1.26). See [S1].
  - `GOEXPERIMENT=jsonv2` for compatibility/perf testing only, never as the default build until stabilized. See [S2].

Forbidden
- Shipping a production build that requires `GOEXPERIMENT=*` to function.
- Silencing compatibility or perf regressions by pinning an old toolchain forever.

## 2. Documentation rules (the approved generator is the source of truth)

Why this rule exists: Agents generate lots of code. If docs are not co-located and enforced, they rot immediately. See [S29].

2.1 Doc comment format.
- Every exported package MUST have a `doc.go` with a package comment explaining: purpose, invariants, concurrency model, and performance sensitivities.
- Every exported type/function MUST have a doc comment that starts with its name and states:
  1) ownership/concurrency expectations,
  2) allocation/perf notes if it is on a hot path,
  3) error semantics.

2.2 README generation is required and enforced.
- Each package directory MAY contain a generated `README.md`.
- Root `README.md` MUST be generated.
- CI MUST run the approved documentation generator and fail if generated output differs.

Reference command pattern (pin a version for reproducibility):
```bash
go run <approved-doc-generator>@<pinned> -cmd -all -o README.md .
```
See [S29].

2.3 Generated docs do not get hand-edited.
- If you want to change docs, change doc comments, then regenerate.

Forbidden
- Hand-editing generated README output.
- Writing “explainer” docs that duplicate the code without stating invariants, ownership, and costs.

## 3. Mandatory state machines with the approved HSM library

Why this rule exists: Complex stateful systems fail from implicit state spread across goroutines. An HSM makes state explicit, testable, and controllable without mutex soup. See [S30], [S16].

### 3.1 When you MUST use an HSM
You MUST model the behavior as an HSM when any of these are true:
1) Multi-step behavior across time (retries, backoff, timeouts).
2) Protocol handlers (network, stream, file formats, workflows).
3) Lifecycle management (start/stop/restart; warmup; draining).
4) Orchestration (fan-out/fan-in; sagas; compensations).
5) Anything with more than 2 boolean flags worth of “state”.

### 3.2 HSM structure rules (performance + determinism)
3.2.1 One state machine instance owns its mutable state.
- All mutable fields that represent machine state live on the HSM instance struct.
- Only the machine’s operations mutate them.

3.2.2 Keep transitions pure, push side effects into operations.
- Guards MUST be pure (no I/O, no logging, no allocations beyond trivial).
- Transition actions SHOULD schedule side effects via operations that are executed by the machine runtime, not inline by random goroutines.

3.2.3 Event-driven, bounded input.
- External inputs must be converted into HSM events and dispatched.
- If you queue events via channels, channels MUST be bounded and MUST implement backpressure or load shedding.

3.2.4 Time is modeled as events.
- Timeouts, intervals, and backoff MUST be represented as timer events (not scattered `time.After` calls). See timer rules in §4.7.

3.2.5 State naming is hierarchical and stable.
- Use `Subsystem.State` naming (example: `Conn.Open`, `Conn.Draining`, `Conn.Closed`).
- Event names MUST be stable strings and versionable (example: `conn.open`, `conn.rx`, `conn.timeout`).

3.2.6 Deterministic tests.
- Use `testing/synctest` for time-based and concurrent state machine tests when feasible. See [S16], [S2].

Short model example (structure only):
```go
model := hsm.Define(
  "conn",
  hsm.State("Conn.Closed"),
  hsm.State("Conn.Open"),
  hsm.Transition(hsm.Trigger("conn.open"), hsm.Source("Conn.Closed"), hsm.Target("Conn.Open")),
  hsm.Initial("Conn.Closed"),
)
sm := hsm.Start(context.Background(), &ConnHSM{}, &model)
_ = hsm.Dispatch(context.Background(), sm, hsm.Event{Name: "conn.open"})
```
See [S30].

### 3.3 HSM performance rules
3.3.1 Hot operations must be allocation-aware.
- Avoid reflection and `fmt` in operations on hot paths.
- Any new allocations in hot operations require a benchmark showing impact. See §4.8.

3.3.2 Never block the machine on unbounded I/O.
- Long I/O must be moved to owned worker goroutines that report results back as events.

3.3.3 Avoid cross-machine shared locks.
- If multiple machines need shared data, use message passing or sharded ownership (§5.3) rather than a global lock.

Forbidden
- “State machine” implemented as a switch statement plus goroutines and shared mutable fields.
- Blocking the HSM event loop on network calls, disk I/O, or waiting on another subsystem lock.

## 4. Performance and memory (CPU, allocations, GC, latency)

Why this rule exists: In Go 1.26, the default GC is faster, but allocations and long-lived pointers still dominate tail latency and throughput. Measure, then change. See [S1], [S5], [S15].

### 4.1 Measure-first rules
4.1.1 No performance claims without a measurement.
- For any “optimization” PR, include at least one of: benchmark (`testing.B`), pprof screenshot, or trace snippet.

4.1.2 Hot path identification is required.
- Mark hot functions with a short comment: `// HOT: <reason> (pps|qps|tail-latency)`.

### 4.2 Allocation minimization (strings, bytes, buffers)
4.2.1 Prefer streaming APIs.
- Prefer `io.Reader`/`io.Writer` streaming to building giant `[]byte` or `string` blobs.

4.2.2 Build strings with `strings.Builder` or `bytes.Buffer`, not repeated concatenation.

4.2.3 Avoid `fmt` in hot paths.
- Prefer `strconv.Append*`, `strconv.Format*`, `encoding/hex`, `encoding/base64` for conversions.

4.2.4 Avoid `any` in hot paths.
- Using `any` tends to allocate and trigger reflection in downstream code (including logging).

Forbidden
- `fmt.Sprintf` inside request loops.
- Converting `[]byte` to `string` repeatedly in tight loops.

### 4.3 Escape analysis awareness (make the compiler keep data on stack)
4.3.1 If it is hot, check escapes.
- For hot packages, CI SHOULD run: `go test -c -gcflags=all=-m=2` (or equivalent) and surface new heap escapes in PR review.

4.3.2 Prefer value ownership.
- Prefer passing structs by value when they are small and immutable.
- Avoid returning pointers to short-lived locals from hot functions.

4.3.3 Do not hide allocations behind interfaces.
- Do not store concrete values into `interface{}`/`any` in hot paths unless benchmarked.

Note: Go 1.26 improves stack allocation in more cases (including slice backing store in more situations). Still, you must verify escapes in your code. See [S1].

Forbidden
- “It probably doesn’t allocate” without checking.

### 4.4 GC pressure rules
4.4.1 Prefer fewer long-lived objects over fewer total objects.
- Long-lived heap with pointers increases marking work and tail latency. Tune lifetimes and retention first. See [S5].

4.4.2 Do not retain large backing arrays.
- When slicing large buffers, copy out what you keep long-term.

4.4.3 Set a memory limit in containerized environments where OOM kills are possible.
- Use `GOMEMLIMIT` or `runtime/debug.SetMemoryLimit` with a documented rationale. See [S5], [S8].

4.4.4 Treat `sync.Pool` as a cache, not an ownership mechanism.
- Anything in a `sync.Pool` may disappear at any time. Do not pool objects that must be closed, finalized, or deterministically freed. See [S6].

Forbidden
- Using `sync.Pool` for “singletons” or resources with cleanup (files, sockets, goroutines). See [S6].

### 4.5 Slice and map capacity planning
4.5.1 Always pre-size when you have a bound.
- `make([]T, 0, n)` when `n` is known or bounded.
- `make(map[K]V, n)` when `n` is known or bounded.

4.5.2 Don’t grow buffers by tiny increments.
- Growth policies must be geometric for amortized O(1).

4.5.3 Avoid per-request map churn.
- If a request needs a scratch map, prefer a fixed struct, small slice, or a pooled map with careful reset and measured benefit.

Forbidden
- `append` in a tight loop with a known final size and no preallocation.

### 4.6 Zero-value and struct layout
4.6.1 Zero value must be usable.
- Types must behave safely and predictably when zero-initialized (no required Init unless strictly necessary). See [S32].

4.6.2 Struct layout is a performance choice.
- Group fields to reduce padding (largest to smallest) when the struct is frequently allocated or copied.
- Separate hot fields from cold fields (hot struct embedded in cold struct) to improve cache locality.

Forbidden
- “Initializer required” types for common containers or concurrency primitives.

### 4.7 Timers, tickers, and time
4.7.1 Do not allocate timers in loops.
- Avoid `time.After` in loops and hot paths. Prefer `time.NewTimer` and `Reset` with careful `Stop`/drain.

4.7.2 Understand version-dependent timer semantics.
- Go 1.23+ makes unused timers and tickers eligible for GC even if unstopped (with module `go` line 1.23.0+). This reduces leaks, but it does not make timer allocation free. See [S3].

4.7.3 Time control for tests.
- Use `testing/synctest` for time-based concurrency tests where applicable. See [S16].

Forbidden
- `time.Sleep` for synchronization in tests.

### 4.8 Benchmarking and baselines (`testing.B` only)
4.8.1 Every hot change needs a benchmark.
- Add or update `BenchmarkXxx` in the same package.

4.8.2 Benchmarks must report allocations.
- Call `b.ReportAllocs()` in all microbenchmarks.

4.8.3 Benchmarks must isolate setup.
- Use `b.StopTimer` / `b.StartTimer` / `b.ResetTimer`.

4.8.4 Baseline discipline.
- If a benchmark regresses >5% in ns/op or allocs/op on the same hardware class, the PR must explain why and what mitigations exist.

Reference docs: `testing` and `runtime/pprof` integration with `go test` profiling flags. See [S14], [S15].

Forbidden
- Comparing benchmarks across different machines without controlling environment.

### 4.9 Profiling and diagnostics (stdlib-first)
4.9.1 Use the official diagnostics workflow.
- Prefer `pprof`, execution traces, and runtime metrics before guessing. See [S15].

4.9.2 Prefer built-in profiling endpoints.
- Use `net/http/pprof` for servers (debug-only exposure). See [S13].

4.9.3 Use runtime metrics for lightweight telemetry.
- Collect `/gc/*` and scheduler metrics via `runtime/metrics` for GC and load insights. See [S7].

4.9.4 Use trace selectively.
- Prefer `runtime/trace` for scheduler and blocking issues, and the Go flight recorder when you need always-on recent-history traces. See [S24], [S23].

Forbidden
- Adding an external profiling agent before using pprof/trace/runtime metrics.

## 5. Concurrency rules (structured, scalable, no hand-waving)

Why this rule exists: Go makes concurrency easy to start and easy to leak. The rules below keep ownership explicit, avoid contention, and preserve determinism. See [S1], [S4], [S16], [S32].

### 5.1 Structured concurrency (goroutine lifecycle)
5.1.1 Every goroutine must have:
- an owner (the function or component responsible),
- a stop signal (context cancellation or a closed channel),
- a join point (WaitGroup or equivalent),
- a shutdown deadline.

5.1.2 Goroutines are not fire-and-forget.
- If a goroutine outlives the request/component that spawned it, it must be managed by a long-lived supervisor (often an HSM) with explicit start/stop.

5.1.3 Prefer `sync.WaitGroup.Go` for simple fan-out.
- When you only need “spawn and wait”, use the Go 1.25 `WaitGroup.Go` helper to reduce common `Add` misuse and vet warnings. See [S2].

Forbidden
- Spawning goroutines inside loops without bounding or without a join.
- A goroutine that can block forever on send/recv without honoring cancellation.

### 5.2 Context propagation
5.2.1 Context is for cancellation and deadlines.
- `context.Context` must be the first parameter, named `ctx`.
- Do not store contexts in structs; pass them through the call chain. See [S10], [S11].

5.2.2 Timeouts belong at boundaries.
- Add timeouts at ingress (HTTP/RPC) and on outbound I/O calls.
- Do not create new contexts in hot inner loops.

5.2.3 Context values are for request-scoped metadata only.
- Values must be low-cardinality and stable (request id, trace id), not bulky structs.

Forbidden
- `context.WithValue` used for optional parameters or dependency injection.

### 5.3 Ownership models (avoid contention by design)
5.3.1 Single-writer ownership.
- If a data structure is mutated frequently, designate a single goroutine as the owner and interact via message passing.

5.3.2 Sharded ownership.
- For high-throughput keyed state (caches, sessions), shard by key hash into N owners.
- Each shard owns its map and processes requests sequentially (actor-like) to avoid lock contention.

5.3.3 Per-key partitioning.
- If operations are per-key independent, route work by key so that the same key always hits the same shard.

5.3.4 Cross-shard coordination.
- Do not lock across shards. Use higher-level events, versioning, or two-phase operations in an orchestrating HSM.

Forbidden
- One global mutex protecting a map that is touched on every request.

### 5.4 Channel patterns (bounded, cancelable, backpressure-aware)
5.4.1 Channels must be bounded unless proven otherwise.
- Buffer size must be justified (throughput, latency, memory).

5.4.2 Backpressure is required.
- When the channel is full, choose one:
  1) block with a context deadline,
  2) drop and count (load shedding),
  3) spill to disk (rare, must be explicit).

5.4.3 Cancellation-safe sends.
- Any send that might block must select on `ctx.Done()`.

Pattern:
```go
select {
case q <- msg:
  return nil
case <-ctx.Done():
  return ctx.Err()
}
```

5.4.4 Closing channels.
- Only the sender (owner) closes the channel.
- Receivers must treat closure as terminal state.

Forbidden
- Unbounded queues implemented as `for { ch <- ... }` with no deadline.

### 5.5 When locks are acceptable (and when to redesign)
5.5.1 Locks are acceptable when:
- critical sections are tiny,
- contention is low (measured via mutex profile),
- the lock does not cross I/O boundaries.

5.5.2 Redesign is required when:
- lock contention shows up in pprof,
- tail latency is impacted,
- the lock is held during I/O or calls out to unknown code.

5.5.3 Prefer simpler locks.
- Prefer `sync.Mutex` over `sync.RWMutex` unless read ratio is extreme and measured.

5.5.4 Use atomics for counters, not for complex invariants.
- Atomics are for simple numeric state and pointers, not multi-field consistency.

Forbidden
- Using RWMutex “because reads are common” without profiling.

### 5.6 Rate limiting and load shedding (stdlib-first)
5.6.1 Rate limiting must be local and cheap.
- Use a token bucket implemented with a bounded channel and ticker, or a fixed window counter, depending on requirements.

5.6.2 Shed load before you melt.
- When saturated, reject early (for servers: return a clear error code) and record metrics.

Sketch (token bucket):
```go
// fill tokens at rate r, capacity cap.
tokens := make(chan struct{}, cap)
ticker := time.NewTicker(time.Second / time.Duration(r))
// in owner goroutine: on each tick, try to add a token (non-blocking)
```

Forbidden
- Creating one goroutine per request just to sleep for rate limiting.

### 5.7 Leak detection and reproducibility
5.7.1 Use synctest for deterministic concurrent tests.
- Prefer virtual time and bubble isolation to real sleeps. See [S16].

5.7.2 Use the Go 1.26 goroutine leak profile in CI for leak hunts.
- In dedicated CI jobs, build with `GOEXPERIMENT=goroutineleakprofile` and collect `/debug/pprof/goroutineleak` when tests fail or hang. See [S1].

Forbidden
- Ignoring goroutine leaks because “the process exits anyway”.

## 6. Error handling rules

Why this rule exists: In concurrent systems, ambiguous errors cause retries, log storms, and misclassification. Errors must be actionable and composable. See [S12], [S32].

6.1 Wrap, do not rewrite.
- When adding context, wrap with `%w` so callers can use `errors.Is/As`.

6.2 Sentinel vs typed errors.
- Use sentinel errors for stable, comparable conditions (`var ErrNotFound = errors.New(...)`).
- Use typed errors when callers need structured fields.

6.3 Taxonomy.
- Domain errors: stable semantics, safe to expose.
- Transport errors: network/timeouts; include operation and endpoint.
- Programmer errors: invariant violations; use panic only if continuing would corrupt state.

6.4 Join errors instead of losing them.
- Use `errors.Join` to return multiple independent errors (shutdown cleanup, fan-in). See [S12].

6.5 Do not log and return the same error.
- Either log at the boundary (where you decide to drop/return) or return up the stack, not both.

6.6 Context errors.
- Treat `context.Canceled` and `context.DeadlineExceeded` as control flow, not “errors worth paging”.

Forbidden
- `panic` for expected runtime failures (I/O, validation).
- Returning raw `fmt.Errorf("%v", err)` without `%w` when propagation is needed.

## 7. Structured logging (standard library first)

Why this rule exists: Logs are a performance cost and an incident response tool. Structured logs enable low-cost filtering and correlation. See [S9].

7.1 Use `log/slog`.
- Default logger must be `slog.Logger` with a JSON handler in production.

7.2 Required fields (minimum set).
Every log record in service code MUST include these keys (directly or via logger scope):
- `component` (stable subsystem name)
- `operation` (stable verb/noun)
- `request_id` (if request-scoped)
- `trace_id` / `span_id` (if tracing is enabled)
- `duration_ms` (for completed operations)
- `error` (when non-nil)

7.3 Level rules.
- Debug: high-volume, disabled by default.
- Info: lifecycle, steady-state summaries.
- Warn: degraded behavior, retries, shedding.
- Error: request failure, invariant breach that still allows continuation.

7.4 Performance rules.
- Prefer typed attrs (`slog.String`, `slog.Int64`, etc.) over `slog.Any` on hot paths to avoid reflection cost. See [S9].
- Do not build log strings eagerly. Use structured fields.

7.5 Sampling.
- Debug logs MUST be sampled or gated in hot paths.
- Implement sampling as a lightweight handler wrapper (no external deps).

Forbidden
- Printf-style logging (`log.Printf`, `fmt.Printf`) in production request paths.
- Logging inside tight loops without sampling.

## 8. Telemetry (metrics, traces, profiling)

Why this rule exists: You cannot govern performance without observability. Use the smallest tool that answers the question. See [S15].

### 8.1 Profiling expectations
8.1.1 pprof endpoints are available in non-prod or behind auth.
- `net/http/pprof` is the default for servers. See [S13].

8.1.2 Use `go tool pprof` and keep profiles small.
- For benchmarks, use `go test -cpuprofile/-memprofile` paths. See [S14].

8.1.3 Prefer the Go flight recorder for always-on traces.
- Use the built-in flight recorder API introduced in Go 1.25 to capture recent trace history with low overhead. See [S23], [S2].

### 8.2 Runtime metrics
8.2.1 Use `runtime/metrics` for low-overhead counters.
- Export key GC and scheduler metrics (`/gc/*`, goroutines, etc.) via your existing metrics sink. See [S7].

### 8.3 OpenTelemetry (allowed only with strict justification)
Default rule: Do not add OpenTelemetry to a project unless you need cross-service tracing or vendor-neutral telemetry export.

If OTel is approved:
8.3.1 Boundaries.
- Instrument only at subsystem boundaries (ingress/egress, queue boundaries). Avoid per-function spans.
- For libraries: depend on OTel API only, not the SDK, so the application controls exporters. See [S25].

8.3.2 Cardinality control is mandatory.
- Metrics attributes that can be high-cardinality must be opt-in by convention; avoid user IDs, request IDs, full URLs in metrics. See [S26].
- Configure an explicit metric cardinality limit (`WithCardinalityLimit`) and document it. See [S27].

8.3.3 Cost control.
- Sampling decisions must be explicit for traces.
- Do not emit debug-level logs via OTel logs signal unless approved (logs signal remains experimental in OTel docs). See [S25].

Forbidden
- Adding OTel “everywhere” because it is easy.
- High-cardinality attributes on metrics without an enforced limit.

## 9. Naming conventions

Why this rule exists: Naming is a performance and correctness tool for agents. Consistent names reduce accidental misuse and make reviews faster.

9.1 Packages.
- Lowercase, no underscores, short, and non-stuttering (`httpclient`, not `http_client` or `clienthttp`).
- `internal/` packages are for non-public APIs. See [S31].

9.2 Files.
- `foo.go` for main implementation, `foo_test.go` for tests, `doc.go` for package docs.

9.3 Types and methods.
- Exported: `CamelCase`.
- Interfaces: keep small and consumer-defined (“the bigger the interface, the weaker the abstraction”). See [S32].

9.4 Receivers.
- One or two letters, derived from type name (`c *Conn`, `sm *ServerMachine`).
- Do not use `this`, `self`.

9.5 Error names.
- Sentinel errors: `var ErrXxx = errors.New("...")`.
- Typed errors: `type XxxError struct { ... }`.

9.6 HSM naming.
- States: `Subsystem.State` (stable strings).
- Events: `subsystem.verb` or `subsystem.noun.verb`.
- Operations: verbs (`Dial`, `Flush`, `Commit`, `Abort`).

Forbidden
- `IService`, `ManagerManager`, `Util`, `Common` package names.

## 10. Package layout and boundaries

Why this rule exists: Dependency direction is a performance tool. Clear boundaries prevent cyclic imports, lock coupling, and accidental hot-path bloat.

10.1 Layout for medium/large projects.
- `/cmd/<app>` for binaries, minimal logic.
- `/internal/<subsystem>` for implementation.
- `/pkg/<name>` only for intentionally public, versioned libraries.

10.2 Internal packages enforce boundaries.
- Code under `internal/` must not be imported from outside its parent tree. See [S31].

10.3 Avoid cycles by construction.
- Dependencies flow inward: `cmd -> internal/app -> internal/subsystems -> internal/core`.
- Shared types go in the lowest-level package that does not depend upward.

10.4 HSM boundaries align to subsystems.
- Each subsystem that has state owns one HSM.
- Cross-subsystem orchestration uses a separate orchestrator HSM, not shared mutable state.

Forbidden
- “utils” packages that become dumping grounds.
- Import cycles fixed by moving code into `internal/common`.

## 11. Testing (builtin only)

Why this rule exists: Performance and concurrency regressions are bugs. Tests must be deterministic and fast enough to run constantly.

11.1 Unit tests are table-driven by default.
- Use subtests (`t.Run`) for cases.

11.2 Determinism rules.
- No `time.Sleep` for ordering.
- Fake time or use `testing/synctest` for concurrent/time-based logic. See [S16].
- Randomness must be seeded deterministically in tests.

11.3 Concurrency diagnostics.
- The canonical CI/quality gate must run with `CGO_ENABLED=0` and must not require `-race`.
- Race-detector runs are optional developer diagnostics only, explicitly CGO-enabled, and must not block merge or release gates.

11.4 Fuzzing rules (builtin).
- Add fuzz tests for parsers, decoders, and protocol inputs.
- Use `go test -fuzz=FuzzXxx` in CI on a schedule, and check in interesting corpora. See [S17].

11.5 Benchmarks and perf regression gates.
- Use `go test -bench` in CI for critical packages.
- Use `-benchmem` and/or `b.ReportAllocs()`.

Forbidden
- Flaky tests justified as “timing issues”. Fix them.
- External assertion frameworks.

## 12. Review checklist (must be satisfied)

12.1 Dependency check.
- No new module dependencies beyond allowed set (§0.2).

12.2 Concurrency check.
- Every goroutine has owner/stop/join.
- No unbounded queues.

12.3 Performance check.
- Hot changes have benchmarks and report allocations.
- No new `fmt` or reflection in hot paths.

12.4 Memory check.
- No `sync.Pool` misuse.
- No new long-lived retention of large buffers.

12.5 HSM check.
- Any new multi-step behavior is modeled in the approved HSM library with explicit events and operations.

12.6 Docs check.
- Doc comments updated.
- Generated documentation output has been refreshed and matches.


## Additional Merged Rules

- GO-0.1: No CGO. Build and test with `CGO_ENABLED=0`. Any PR that introduces `import "C"`, `// #cgo`, or requires a C toolchain is rejected. (We do not “make an exception” because the runtime performance and portability trade-offs leak everywhere.) See [S32].
- GO-0.2: Dependencies must be minimal.
- GO-0.2.1: Prefer standard library.
- GO-0.2.2: Allowed third-party runtime state-machine dependency: a single approved HSM library only, and only for explicitly modeled stateful behavior. See [S30].
- GO-0.2.3: Allowed third-party documentation tool dependency: a single approved README generator only, when documentation generation is required. See [S29].
- GO-0.2.4: Any other dependency requires explicit written justification in the PR description (what stdlib option failed, measurable cost/benefit, and a removal plan). Permission for such dependencies may be explicitly asked for or granted in requirements.
- GO-0.3: Testing must use only the builtin `testing` package. No external test frameworks. See [S14], [S16], [S17].
- GO-0.4: Docs live with the code. If README generation is part of the project standard, generated READMEs must match the checked-in output. See [S29].
- GO-0.5: Stateful behavior MUST use the project-approved HSM library.
- GO-FORB-001: Adding a new concurrency-heavy workflow using ad-hoc goroutines, callback chains, or implicit state in scattered booleans.
- GO-FORB-002: “Just make it single threaded” as a performance workaround.
- GO-1.1: Pin the toolchain. `go.mod` MUST include a `toolchain` line pinned to an exact patch version (example: `toolchain go1.26.0`) and a `go` line pinned to a Go language version used by the project (example: `go 1.26.0`). See [S19], [S20].
- GO-1.2: Upgrade policy. When upgrading Go, run `go fix` and commit the resulting changes as a standalone PR (no behavioral changes mixed in). See [S28].
- GO-1.3: Container CPU limits. Do not vendor or depend on `automaxprocs`. Go 1.25+ is container-aware by default; do not override `GOMAXPROCS` unless you have measured tail-latency impact for a specific workload. See [S4], [S2].
- GO-1.4: Experiments policy. `GOEXPERIMENT` MUST be empty in production builds unless the project explicitly opts in. Allowed in CI-only builds: `GOEXPERIMENT=goroutineleakprofile` for leak detection (Go 1.26). See [S1]. `GOEXPERIMENT=jsonv2` for compatibility/perf testing only, never as the default build until stabilized. See [S2].
- GO-FORB-003: Shipping a production build that requires `GOEXPERIMENT=*` to function.
- GO-FORB-004: Silencing compatibility or perf regressions by pinning an old toolchain forever.
- GO-2.1: Doc comment format. Every exported package MUST have a `doc.go` with a package comment explaining: purpose, invariants, concurrency model, and performance sensitivities. Every exported type/function MUST have a doc comment that starts with its name and states:
- GO-2.1.1: ownership/concurrency expectations,
- GO-2.1.2: allocation/perf notes if it is on a hot path,
- GO-2.1.3: error semantics.
- GO-2.2: README generation is required and enforced. Each package directory MAY contain a generated `README.md`. Root `README.md` MUST be generated. CI MUST run the approved documentation generator and fail if generated output differs.
- GO-2.3: Generated docs do not get hand-edited. If you want to change docs, change doc comments, then regenerate.
- GO-FORB-005: Hand-editing generated README output.
- GO-FORB-006: Writing “explainer” docs that duplicate the code without stating invariants, ownership, and costs.
- GO-2.3.1: Multi-step behavior across time (retries, backoff, timeouts).
- GO-2.3.2: Protocol handlers (network, stream, file formats, workflows).
- GO-2.3.3: Lifecycle management (start/stop/restart; warmup; draining).
- GO-2.3.4: Orchestration (fan-out/fan-in; sagas; compensations).
- GO-2.3.5: Anything with more than 2 boolean flags worth of “state”.
- GO-3.2.1: One state machine instance owns its mutable state. All mutable fields that represent machine state live on the HSM instance struct. Only the machine’s operations mutate them.
- GO-3.2.2: Keep transitions pure, push side effects into operations. Guards MUST be pure (no I/O, no logging, no allocations beyond trivial). Transition actions SHOULD schedule side effects via operations that are executed by the machine runtime, not inline by random goroutines.
- GO-3.2.3: Event-driven, bounded input. External inputs must be converted into HSM events and dispatched. If you queue events via channels, channels MUST be bounded and MUST implement backpressure or load shedding.
- GO-3.2.4: Time is modeled as events. Timeouts, intervals, and backoff MUST be represented as timer events (not scattered `time.After` calls). See timer rules in §4.7.
- GO-3.2.5: State naming is hierarchical and stable. Use `Subsystem.State` naming (example: `Conn.Open`, `Conn.Draining`, `Conn.Closed`). Event names MUST be stable strings and versionable (example: `conn.open`, `conn.rx`, `conn.timeout`).
- GO-3.2.6: Deterministic tests. Use `testing/synctest` for time-based and concurrent state machine tests when feasible. See [S16], [S2].
- GO-3.3.1: Hot operations must be allocation-aware. Avoid reflection and `fmt` in operations on hot paths. Any new allocations in hot operations require a benchmark showing impact. See §4.8.
- GO-3.3.2: Never block the machine on unbounded I/O. Long I/O must be moved to owned worker goroutines that report results back as events.
- GO-3.3.3: Avoid cross-machine shared locks. If multiple machines need shared data, use message passing or sharded ownership (§5.3) rather than a global lock.
- GO-FORB-007: “State machine” implemented as a switch statement plus goroutines and shared mutable fields.
- GO-FORB-008: Blocking the HSM event loop on network calls, disk I/O, or waiting on another subsystem lock.
- GO-4.1.1: No performance claims without a measurement. For any “optimization” PR, include at least one of: benchmark (`testing.B`), pprof screenshot, or trace snippet.
- GO-4.1.2: Hot path identification is required. Mark hot functions with a short comment: `// HOT: <reason> (pps|qps|tail-latency)`.
- GO-4.2.1: Prefer streaming APIs. Prefer `io.Reader`/`io.Writer` streaming to building giant `[]byte` or `string` blobs.
- GO-4.2.2: Build strings with `strings.Builder` or `bytes.Buffer`, not repeated concatenation.
- GO-4.2.3: Avoid `fmt` in hot paths. Prefer `strconv.Append*`, `strconv.Format*`, `encoding/hex`, `encoding/base64` for conversions.
- GO-4.2.4: Avoid `any` in hot paths. Using `any` tends to allocate and trigger reflection in downstream code (including logging).
- GO-FORB-009: `fmt.Sprintf` inside request loops.
- GO-FORB-010: Converting `[]byte` to `string` repeatedly in tight loops.
- GO-4.3.1: If it is hot, check escapes. For hot packages, CI SHOULD run: `go test -c -gcflags=all=-m=2` (or equivalent) and surface new heap escapes in PR review.
- GO-4.3.2: Prefer value ownership. Prefer passing structs by value when they are small and immutable. Avoid returning pointers to short-lived locals from hot functions.
- GO-4.3.3: Do not hide allocations behind interfaces. Do not store concrete values into `interface{}`/`any` in hot paths unless benchmarked.
- GO-FORB-011: “It probably doesn’t allocate” without checking.
- GO-4.4.1: Prefer fewer long-lived objects over fewer total objects. Long-lived heap with pointers increases marking work and tail latency. Tune lifetimes and retention first. See [S5].
- GO-4.4.2: Do not retain large backing arrays. When slicing large buffers, copy out what you keep long-term.
- GO-4.4.3: Set a memory limit in containerized environments where OOM kills are possible. Use `GOMEMLIMIT` or `runtime/debug.SetMemoryLimit` with a documented rationale. See [S5], [S8].
- GO-4.4.4: Treat `sync.Pool` as a cache, not an ownership mechanism. Anything in a `sync.Pool` may disappear at any time. Do not pool objects that must be closed, finalized, or deterministically freed. See [S6].
- GO-FORB-012: Using `sync.Pool` for “singletons” or resources with cleanup (files, sockets, goroutines). See [S6].
- GO-4.5.1: Always pre-size when you have a bound. `make([]T, 0, n)` when `n` is known or bounded. `make(map[K]V, n)` when `n` is known or bounded.
- GO-4.5.2: Don’t grow buffers by tiny increments. Growth policies must be geometric for amortized O(1).
- GO-4.5.3: Avoid per-request map churn. If a request needs a scratch map, prefer a fixed struct, small slice, or a pooled map with careful reset and measured benefit.
- GO-FORB-013: `append` in a tight loop with a known final size and no preallocation.
- GO-4.6.1: Zero value must be usable. Types must behave safely and predictably when zero-initialized (no required Init unless strictly necessary). See [S32].
- GO-4.6.2: Struct layout is a performance choice. Group fields to reduce padding (largest to smallest) when the struct is frequently allocated or copied. Separate hot fields from cold fields (hot struct embedded in cold struct) to improve cache locality.
- GO-FORB-014: “Initializer required” types for common containers or concurrency primitives.
- GO-4.7.1: Do not allocate timers in loops. Avoid `time.After` in loops and hot paths. Prefer `time.NewTimer` and `Reset` with careful `Stop`/drain.
- GO-4.7.2: Understand version-dependent timer semantics. Go 1.23+ makes unused timers and tickers eligible for GC even if unstopped (with module `go` line 1.23.0+). This reduces leaks, but it does not make timer allocation free. See [S3].
- GO-4.7.3: Time control for tests. Use `testing/synctest` for time-based concurrency tests where applicable. See [S16].
- GO-FORB-015: `time.Sleep` for synchronization in tests.
- GO-4.8.1: Every hot change needs a benchmark. Add or update `BenchmarkXxx` in the same package.
- GO-4.8.2: Benchmarks must report allocations. Call `b.ReportAllocs()` in all microbenchmarks.
- GO-4.8.3: Benchmarks must isolate setup. Use `b.StopTimer` / `b.StartTimer` / `b.ResetTimer`.
- GO-4.8.4: Baseline discipline. If a benchmark regresses >5% in ns/op or allocs/op on the same hardware class, the PR must explain why and what mitigations exist.
- GO-FORB-016: Comparing benchmarks across different machines without controlling environment.
- GO-4.9.1: Use the official diagnostics workflow. Prefer `pprof`, execution traces, and runtime metrics before guessing. See [S15].
- GO-4.9.2: Prefer built-in profiling endpoints. Use `net/http/pprof` for servers (debug-only exposure). See [S13].
- GO-4.9.3: Use runtime metrics for lightweight telemetry. Collect `/gc/*` and scheduler metrics via `runtime/metrics` for GC and load insights. See [S7].
- GO-4.9.4: Use trace selectively. Prefer `runtime/trace` for scheduler and blocking issues, and the Go flight recorder when you need always-on recent-history traces. See [S24], [S23].
- GO-FORB-017: Adding an external profiling agent before using pprof/trace/runtime metrics.
- GO-5.1.1: Every goroutine must have: an owner (the function or component responsible), a stop signal (context cancellation or a closed channel), a join point (WaitGroup or equivalent), a shutdown deadline.
- GO-5.1.2: Goroutines are not fire-and-forget. If a goroutine outlives the request/component that spawned it, it must be managed by a long-lived supervisor (often an HSM) with explicit start/stop.
- GO-5.1.3: Prefer `sync.WaitGroup.Go` for simple fan-out. When you only need “spawn and wait”, use the Go 1.25 `WaitGroup.Go` helper to reduce common `Add` misuse and vet warnings. See [S2].
- GO-FORB-018: Spawning goroutines inside loops without bounding or without a join.
- GO-FORB-019: A goroutine that can block forever on send/recv without honoring cancellation.
- GO-5.2.1: Context is for cancellation and deadlines. `context.Context` must be the first parameter, named `ctx`. Do not store contexts in structs; pass them through the call chain. See [S10], [S11].
- GO-5.2.2: Timeouts belong at boundaries. Add timeouts at ingress (HTTP/RPC) and on outbound I/O calls. Do not create new contexts in hot inner loops.
- GO-5.2.3: Context values are for request-scoped metadata only. Values must be low-cardinality and stable (request id, trace id), not bulky structs.
- GO-FORB-020: `context.WithValue` used for optional parameters or dependency injection.
- GO-5.3.1: Single-writer ownership. If a data structure is mutated frequently, designate a single goroutine as the owner and interact via message passing.
- GO-5.3.2: Sharded ownership. For high-throughput keyed state (caches, sessions), shard by key hash into N owners. Each shard owns its map and processes requests sequentially (actor-like) to avoid lock contention.
- GO-5.3.3: Per-key partitioning. If operations are per-key independent, route work by key so that the same key always hits the same shard.
- GO-5.3.4: Cross-shard coordination. Do not lock across shards. Use higher-level events, versioning, or two-phase operations in an orchestrating HSM.
- GO-FORB-021: One global mutex protecting a map that is touched on every request.
- GO-5.4.1: Channels must be bounded unless proven otherwise. Buffer size must be justified (throughput, latency, memory).
- GO-5.4.2: Backpressure is required. When the channel is full, choose one:
- GO-5.4.2.1: block with a context deadline,
- GO-5.4.2.2: drop and count (load shedding),
- GO-5.4.2.3: spill to disk (rare, must be explicit).
- GO-5.4.3: Cancellation-safe sends. Any send that might block must select on `ctx.Done()`.
- GO-5.4.4: Closing channels. Only the sender (owner) closes the channel. Receivers must treat closure as terminal state.
- GO-FORB-022: Unbounded queues implemented as `for { ch <- ... }` with no deadline.
- GO-5.5.1: Locks are acceptable when: critical sections are tiny, contention is low (measured via mutex profile), the lock does not cross I/O boundaries.
- GO-5.5.2: Redesign is required when: lock contention shows up in pprof, tail latency is impacted, the lock is held during I/O or calls out to unknown code.
- GO-5.5.3: Prefer simpler locks. Prefer `sync.Mutex` over `sync.RWMutex` unless read ratio is extreme and measured.
- GO-5.5.4: Use atomics for counters, not for complex invariants. Atomics are for simple numeric state and pointers, not multi-field consistency.
- GO-FORB-023: Using RWMutex “because reads are common” without profiling.
- GO-5.6.1: Rate limiting must be local and cheap. Use a token bucket implemented with a bounded channel and ticker, or a fixed window counter, depending on requirements.
- GO-5.6.2: Shed load before you melt. When saturated, reject early (for servers: return a clear error code) and record metrics.
- GO-FORB-024: Creating one goroutine per request just to sleep for rate limiting.
- GO-5.7.1: Use synctest for deterministic concurrent tests. Prefer virtual time and bubble isolation to real sleeps. See [S16].
- GO-5.7.2: Use the Go 1.26 goroutine leak profile in CI for leak hunts. In dedicated CI jobs, build with `GOEXPERIMENT=goroutineleakprofile` and collect `/debug/pprof/goroutineleak` when tests fail or hang. See [S1].
- GO-FORB-025: Ignoring goroutine leaks because “the process exits anyway”.
- GO-6.1: Wrap, do not rewrite. When adding context, wrap with `%w` so callers can use `errors.Is/As`.
- GO-6.2: Sentinel vs typed errors. Use sentinel errors for stable, comparable conditions (`var ErrNotFound = errors.New(...)`). Use typed errors when callers need structured fields.
- GO-6.3: Taxonomy. Domain errors: stable semantics, safe to expose. Transport errors: network/timeouts; include operation and endpoint. Programmer errors: invariant violations; use panic only if continuing would corrupt state.
- GO-6.4: Join errors instead of losing them. Use `errors.Join` to return multiple independent errors (shutdown cleanup, fan-in). See [S12].
- GO-6.5: Do not log and return the same error. Either log at the boundary (where you decide to drop/return) or return up the stack, not both.
- GO-6.6: Context errors. Treat `context.Canceled` and `context.DeadlineExceeded` as control flow, not “errors worth paging”.
- GO-FORB-026: `panic` for expected runtime failures (I/O, validation).
- GO-FORB-027: Returning raw `fmt.Errorf("%v", err)` without `%w` when propagation is needed.
- GO-7.1: Use `log/slog`. Default logger must be `slog.Logger` with a JSON handler in production.
- GO-7.2: Required fields (minimum set).
- GO-7.3: Level rules. Debug: high-volume, disabled by default. Info: lifecycle, steady-state summaries. Warn: degraded behavior, retries, shedding. Error: request failure, invariant breach that still allows continuation.
- GO-7.4: Performance rules. Prefer typed attrs (`slog.String`, `slog.Int64`, etc.) over `slog.Any` on hot paths to avoid reflection cost. See [S9]. Do not build log strings eagerly. Use structured fields.
- GO-7.5: Sampling. Debug logs MUST be sampled or gated in hot paths. Implement sampling as a lightweight handler wrapper (no external deps).
- GO-FORB-028: Printf-style logging (`log.Printf`, `fmt.Printf`) in production request paths.
- GO-FORB-029: Logging inside tight loops without sampling.
- GO-8.1.1: pprof endpoints are available in non-prod or behind auth. `net/http/pprof` is the default for servers. See [S13].
- GO-8.1.2: Use `go tool pprof` and keep profiles small. For benchmarks, use `go test -cpuprofile/-memprofile` paths. See [S14].
- GO-8.1.3: Prefer the Go flight recorder for always-on traces. Use the built-in flight recorder API introduced in Go 1.25 to capture recent trace history with low overhead. See [S23], [S2].
- GO-8.2.1: Use `runtime/metrics` for low-overhead counters. Export key GC and scheduler metrics (`/gc/*`, goroutines, etc.) via your existing metrics sink. See [S7].
- GO-8.3.1: Boundaries. Instrument only at subsystem boundaries (ingress/egress, queue boundaries). Avoid per-function spans. For libraries: depend on OTel API only, not the SDK, so the application controls exporters. See [S25].
- GO-8.3.2: Cardinality control is mandatory. Metrics attributes that can be high-cardinality must be opt-in by convention; avoid user IDs, request IDs, full URLs in metrics. See [S26]. Configure an explicit metric cardinality limit (`WithCardinalityLimit`) and document it. See [S27].
- GO-8.3.3: Cost control. Sampling decisions must be explicit for traces. Do not emit debug-level logs via OTel logs signal unless approved (logs signal remains experimental in OTel docs). See [S25].
- GO-FORB-030: Adding OTel “everywhere” because it is easy.
- GO-FORB-031: High-cardinality attributes on metrics without an enforced limit.
- GO-9.1: Packages. Lowercase, no underscores, short, and non-stuttering (`httpclient`, not `http_client` or `clienthttp`). `internal/` packages are for non-public APIs. See [S31].
- GO-9.2: Files. `foo.go` for main implementation, `foo_test.go` for tests, `doc.go` for package docs.
- GO-9.3: Types and methods. Exported: `CamelCase`. Interfaces: keep small and consumer-defined (“the bigger the interface, the weaker the abstraction”). See [S32].
- GO-9.4: Receivers. One or two letters, derived from type name (`c *Conn`, `sm *ServerMachine`). Do not use `this`, `self`.
- GO-9.5: Error names. Sentinel errors: `var ErrXxx = errors.New("...")`. Typed errors: `type XxxError struct { ... }`.
- GO-9.6: HSM naming. States: `Subsystem.State` (stable strings). Events: `subsystem.verb` or `subsystem.noun.verb`. Operations: verbs (`Dial`, `Flush`, `Commit`, `Abort`).
- GO-FORB-032: `IService`, `ManagerManager`, `Util`, `Common` package names.
- GO-10.1: Layout for medium/large projects. `/cmd/<app>` for binaries, minimal logic. `/internal/<subsystem>` for implementation. `/pkg/<name>` only for intentionally public, versioned libraries.
- GO-10.2: Internal packages enforce boundaries. Code under `internal/` must not be imported from outside its parent tree. See [S31].
- GO-10.3: Avoid cycles by construction. Dependencies flow inward: `cmd -> internal/app -> internal/subsystems -> internal/core`. Shared types go in the lowest-level package that does not depend upward.
- GO-10.4: HSM boundaries align to subsystems. Each subsystem that has state owns one HSM. Cross-subsystem orchestration uses a separate orchestrator HSM, not shared mutable state.
- GO-FORB-033: “utils” packages that become dumping grounds.
- GO-FORB-034: Import cycles fixed by moving code into `internal/common`.
- GO-11.1: Unit tests are table-driven by default. Use subtests (`t.Run`) for cases.
- GO-11.2: Determinism rules. No `time.Sleep` for ordering. Fake time or use `testing/synctest` for concurrent/time-based logic. See [S16]. Randomness must be seeded deterministically in tests.
- GO-11.3: Race detector usage. CI MUST run `go test ./... -race` for packages that use concurrency (or for the whole project where feasible).
- GO-11.4: Fuzzing rules (builtin). Add fuzz tests for parsers, decoders, and protocol inputs. Use `go test -fuzz=FuzzXxx` in CI on a schedule, and check in interesting corpora. See [S17].
- GO-11.5: Benchmarks and perf regression gates. Use `go test -bench` in CI for critical packages. Use `-benchmem` and/or `b.ReportAllocs()`.
- GO-FORB-035: Flaky tests justified as “timing issues”. Fix them.
- GO-FORB-036: External assertion frameworks.
- GO-12.1: Dependency check. No new module dependencies beyond allowed set (§0.2).
- GO-12.2: Concurrency check. Every goroutine has owner/stop/join. No unbounded queues.
- GO-12.3: Performance check. Hot changes have benchmarks and report allocations. No new `fmt` or reflection in hot paths.
- GO-12.4: Memory check. No `sync.Pool` misuse. No new long-lived retention of large buffers.
- GO-12.5: HSM check. Any new multi-step behavior is modeled in the approved HSM library with explicit events and operations.
- GO-12.6: Docs check. Doc comments updated. Generated documentation output has been refreshed and matches.
- GO001: NEVER use CGO; always build and test with `CGO_ENABLED=0`.
- GO002: ALWAYS prefer the standard library over third-party dependencies.
- GO003: ALWAYS use the single approved HSM library for explicitly modeled stateful behavior.
- GO004: ALWAYS use the single approved documentation tool for README generation.
- GO005: ALWAYS provide explicit written justification in the PR description before introducing any other third-party dependency.
- GO006: NEVER use external test frameworks; always use the builtin `testing` package.
- GO007: ALWAYS ensure generated READMEs match the checked-in output when documentation generation is required.
- GO008: ALWAYS model any workflow, lifecycle, protocol handler, orchestration logic, or multi-step behavior as an HSM.
- GO009: NEVER add a new concurrency-heavy workflow using ad-hoc goroutines, callback chains, or implicit state scattered across booleans.
- GO010: NEVER "just make it single threaded" as a performance workaround.
- GO011: ALWAYS pin the toolchain and go version in `go.mod` to an exact patch version.
- GO012: ALWAYS run `go fix` and commit the resulting changes as a standalone PR when upgrading Go.
- GO013: NEVER vendor or depend on `automaxprocs`; do not override `GOMAXPROCS` unless you have measured tail-latency impact.
- GO014: NEVER use `GOEXPERIMENT` in production builds unless the project explicitly opts in.
- GO015: NEVER silence compatibility or performance regressions by pinning an old toolchain forever.
- GO016: ALWAYS include a `doc.go` with a package comment explaining purpose, invariants, concurrency model, and performance sensitivities for every exported package.
- GO017: ALWAYS include a doc comment starting with the type/function name, stating ownership/concurrency expectations, allocation/perf notes, and error semantics for exported symbols.
- GO018: ALWAYS generate the root `README.md` using the approved documentation generator.
- GO019: NEVER hand-edit generated README output.
- GO020: NEVER write "explainer" docs that duplicate the code without stating invariants, ownership, and costs.
- GO021: ALWAYS place all mutable fields representing machine state on the HSM instance struct, modified only by the machine's operations.
- GO022: ALWAYS keep HSM transition guards pure (no I/O, no logging, no allocations beyond trivial).
- GO023: ALWAYS schedule side effects via HSM operations executed by the machine runtime rather than inline by random goroutines.
- GO024: ALWAYS convert external inputs into HSM events and dispatch them.
- GO025: ALWAYS use bounded channels with backpressure or load shedding if queuing events via channels.
- GO026: ALWAYS represent timeouts, intervals, and backoff as timer events rather than scattered `time.After` calls.
- GO027: ALWAYS use hierarchical and stable naming for HSM states (e.g., `Subsystem.State`) and stable, versionable names for events.
- GO028: ALWAYS use `testing/synctest` for time-based and concurrent state machine tests when feasible.
- GO029: NEVER use reflection and `fmt` in operations on hot paths.
- GO030: NEVER block an HSM on unbounded I/O; move long I/O to owned worker goroutines that report results as events.
- GO031: NEVER use cross-machine shared locks; use message passing or sharded ownership instead.
- GO032: NEVER implement a state machine as a switch statement plus goroutines and shared mutable fields.
- GO033: ALWAYS include a benchmark, pprof screenshot, or trace snippet for any optimization PR.
- GO034: ALWAYS mark hot functions with a short comment explaining the reason (e.g., `// HOT: pps`).
- GO035: ALWAYS prefer `io.Reader`/`io.Writer` streaming over building giant byte slices or strings.
- GO036: ALWAYS build strings with `strings.Builder` or `bytes.Buffer` rather than repeated concatenation.
- GO037: NEVER use `fmt` in hot paths; prefer `strconv`, `encoding/hex`, or `encoding/base64`.
- GO038: NEVER use `any` in hot paths to avoid allocations and reflection.
- GO039: NEVER repeatedly convert `[]byte` to `string` in tight loops.
- GO040: ALWAYS verify heap escapes in hot packages using `go test -c -gcflags=all=-m=2`.
- GO041: ALWAYS prefer passing small, immutable structs by value and avoid returning pointers to short-lived locals from hot functions.
- GO042: NEVER store concrete values into `interface{}`/`any` in hot paths without benchmarking.
- GO043: NEVER assume a snippet "probably doesn't allocate" without checking.
- GO044: ALWAYS prefer fewer long-lived objects over fewer total objects to reduce GC marking work.
- GO045: NEVER retain large backing arrays; copy out only the data you need to keep long-term.
- GO046: ALWAYS set a memory limit (`GOMEMLIMIT` or `runtime/debug.SetMemoryLimit`) in containerized environments where OOM kills are possible.
- GO047: NEVER use `sync.Pool` for singletons, or for objects that must be closed, finalized, or deterministically freed.
- GO048: ALWAYS pre-size slices and maps when the capacity is known or bounded.
- GO049: ALWAYS use geometric growth policies for amortized O(1) buffer growth.
- GO050: NEVER create per-request scratch maps; prefer a fixed struct, small slice, or a pooled map.
- GO051: NEVER call `append` in a tight loop with a known final size and no preallocation.
- GO052: ALWAYS ensure the zero value of a type is usable safely and predictably without an explicit initialization method.
- GO053: ALWAYS group struct fields from largest to smallest to reduce padding, and separate hot fields from cold fields.
- GO054: NEVER allocate timers in loops or hot paths with `time.After`; use `time.NewTimer` and `Reset` carefully.
- GO055: NEVER use `time.Sleep` for synchronization in tests; use `testing/synctest` instead.
- GO056: ALWAYS add or update a `testing.B` benchmark for every hot path change.
- GO057: ALWAYS call `b.ReportAllocs()` in all microbenchmarks.
- GO058: ALWAYS isolate setup in benchmarks using `b.StopTimer`, `b.StartTimer`, and `b.ResetTimer`.
- GO059: ALWAYS explain and mitigate any benchmark regression >5% in ns/op or allocs/op in a PR.
- GO060: NEVER compare benchmarks across different machines without a controlled environment.
- GO061: ALWAYS use pprof, execution traces, and runtime metrics before guessing at performance issues.
- GO062: ALWAYS expose `net/http/pprof` for servers in non-prod or debug environments.
- GO063: ALWAYS collect `/gc/*` and scheduler metrics via `runtime/metrics` for load insights.
- GO064: ALWAYS prefer the Go flight recorder API for capturing recent trace history.
- GO065: NEVER add an external profiling agent before using built-in pprof, trace, and runtime metrics.
- GO066: ALWAYS require every goroutine to have an owner, a stop signal, a join point, and a shutdown deadline.
- GO067: ALWAYS use a long-lived supervisor to manage a goroutine that outlives the request or component that spawned it.
- GO068: ALWAYS prefer `sync.WaitGroup.Go` in Go 1.25+ for simple fan-out spawn and wait.
- GO069: NEVER spawn goroutines inside loops without bounding or without a join point.
- GO070: NEVER allow a goroutine to block forever on send/receive without honoring cancellation.
- GO071: ALWAYS make `context.Context` the first parameter named `ctx`, and never store contexts in structs.
- GO072: ALWAYS add context timeouts at ingress and on outbound I/O calls, rather than in hot inner loops.
- GO073: NEVER use `context.WithValue` for optional parameters or dependency injection; use it only for request-scoped metadata.
- GO074: ALWAYS designate a single goroutine as the owner of a frequently mutated data structure and interact via message passing.
- GO075: ALWAYS shard high-throughput keyed state into N owners to process requests sequentially without lock contention.
- GO076: ALWAYS route work by key if operations are per-key independent so the same key hits the same shard.
- GO077: NEVER lock across shards; use higher-level events, versioning, or two-phase operations in an orchestrating HSM.
- GO078: NEVER protect a frequently accessed map with a single global mutex.
- GO079: ALWAYS bound channels and justify their buffer size unless proven otherwise.
- GO080: ALWAYS implement backpressure (blocking with deadline, load shedding, or explicit disk spill) when a channel is full.
- GO081: ALWAYS select on `ctx.Done()` for any channel send that might block.
- GO082: ALWAYS ensure only the sender (owner) closes a channel, and receivers treat closure as terminal state.
- GO083: NEVER implement unbounded queues using an infinite loop without a deadline.
- GO084: ALWAYS restrict locks to tiny critical sections with low contention that do not cross I/O boundaries.
- GO085: ALWAYS redesign if lock contention impacts tail latency or occurs during I/O.
- GO086: NEVER use `sync.RWMutex` simply because reads are common, unless the extreme read ratio is measured and justified; prefer `sync.Mutex`.
- GO087: ALWAYS use atomics only for simple numeric state and pointers, not for multi-field consistency invariants.
- GO088: ALWAYS use local, cheap rate limiting (e.g., a token bucket or fixed window counter).
- GO089: ALWAYS reject requests early and record metrics when saturated to shed load.
- GO090: NEVER create a goroutine per request just to sleep for rate limiting.
- GO091: ALWAYS prefer `testing/synctest` for deterministic concurrent tests over real sleeps.
- GO092: ALWAYS build with `GOEXPERIMENT=goroutineleakprofile` in CI to hunt goroutine leaks on test hangs.
- GO093: NEVER ignore goroutine leaks just because the process will eventually exit.
- GO094: ALWAYS wrap errors with `%w` when adding context so callers can use `errors.Is`/`errors.As`.
- GO095: ALWAYS use sentinel errors for stable, comparable conditions and typed errors for structured fields.
- GO096: ALWAYS use `errors.Join` to return multiple independent errors instead of losing them.
- GO097: NEVER log and return the same error; either log at a boundary or return it up the stack.
- GO098: ALWAYS treat `context.Canceled` and `context.DeadlineExceeded` as control flow rather than actionable errors.
- GO099: NEVER panic for expected runtime failures or return raw `fmt.Errorf` without `%w` if propagation is needed.
- GO100: ALWAYS use `slog.Logger` with a JSON handler in production for structured logging.
- GO101: ALWAYS include component, operation, request_id/trace_id, duration_ms, and error in log records.
- GO102: ALWAYS prefer typed attributes like `slog.String` over `slog.Any` on hot paths to avoid reflection.
- GO103: ALWAYS sample or gate debug logs in hot paths using a lightweight handler wrapper.
- GO104: NEVER use Printf-style logging in production request paths, or log inside tight loops without sampling.
- GO105: ALWAYS make `net/http/pprof` endpoints available in non-prod or behind auth.
- GO106: ALWAYS use `go tool pprof` via `go test -cpuprofile/-memprofile` paths to keep profiles small.
- GO107: ALWAYS prefer the Go flight recorder API for capturing recent trace history.
- GO108: ALWAYS export key GC and scheduler metrics via `runtime/metrics`.
- GO109: NEVER add OpenTelemetry without strict justification, such as cross-service tracing or vendor-neutral export.
- GO110: ALWAYS instrument OpenTelemetry only at subsystem boundaries, avoiding per-function spans.
- GO111: ALWAYS explicitly configure and document a metric cardinality limit (`WithCardinalityLimit`) when using OpenTelemetry.
- GO112: NEVER emit debug-level logs via OpenTelemetry logs signal unless approved, and make sampling decisions explicit for traces.
- GO113: ALWAYS use lowercase, non-stuttering, short names for packages, and `internal/` for non-public APIs.
- GO114: NEVER use `IService`, `ManagerManager`, `Util`, or `Common` for package names.
- GO115: ALWAYS name files with descriptive lowercase names (`foo.go`, `foo_test.go`) and use `doc.go` for package docs.
- GO116: ALWAYS use CamelCase for exported types and keep interfaces small and consumer-defined.
- GO117: ALWAYS use one or two letters derived from the type name
- `go.mod` MUST include a `toolchain` line pinned to an exact patch version (example: `toolchain go1.26.0`) and a `go` line pinned to a Go language version used by the repo (example: `go 1.26.0`). See [S19], [S20].
- `GOEXPERIMENT` MUST be empty in production builds unless the repo explicitly opts in.
- CI MUST run go-docmd and fail if generated output differs.
- CI MUST run `go test ./... -race` for packages that use concurrency (or for the whole repo where feasible).
- Any new multi-step behavior is modeled in `stateforward/hsm.go` with explicit events and operations.
- go-docmd output regenerated and matches.
- GO-2.2: README generation is required and enforced. Each package directory MAY contain a generated `README.md`. Root `README.md` MUST be generated.
- GO-2.2.1: The approved check is `make docs-check` (which runs `go run ./scripts/docgen` and fails if working tree `README.md` differs).
- GO-2.2.2: Docs changes MUST be made in package/module docs (`doc.go`, package comments), then applied via generator; hand-editing generated root README is forbidden.
- GO-12.6: Docs check. Doc comments updated. `make docs-check` must pass (generated output matches repository `README.md`).
- GO-5.2.2: Timeouts belong at boundaries. Add timeouts at ingress (HTTP/RPC) and on outbound I/O calls
- GO-0.5.1: HSM PLANNING GATE — Before planning any component, agents MUST ask: does this component (1) need a mutex, atomics, or any synchronization primitive? (2) act as an actor (owns goroutines, has lifecycle)? (3) consume or produce NATS messages? If ANY answer is YES, plan a proper HSM following the patterns in nats.go/ (hsm.Define, hsm.State, hsm.Entry, etc.). If ALL answers are NO, it stays a plain function/struct.
- GO-0.5.2: Do NOT wrap stateless stores or accessors with HSM layers. The HSM replaces concurrency primitives — it does not add a layer on top of things that don't have them.
- GO-3.2.7: Transient HSM state progression MUST be owned by the machine. Entry, exit, or activity behavior dispatches an event whose kind is `hsm.CompletionEventKind`; service/runtime/adaptor code must not inspect state strings and dispatch the next event.
- GO-3.2.8: HSM entry, exit, activity, effect, and guard behavior MUST NOT call `time.Now()` or other nondeterministic process APIs directly. Use injected dependencies such as `ClockFn`, event-carried values, or HSM timer events.
- GO-3.2.9: Conditional HSM branches MUST be modeled with `hsm.Choice`, pure guards, or ordered guarded transitions. Do not use behavior `if`/`switch`/loop branches to dispatch different transition-driving events when the branch depends on machine state or event payload.
- GO-3.2.10: HSM-owned time MUST be modeled in the graph. Do not use `time.After`, `time.Sleep`, `time.NewTimer`, `time.NewTicker`, `time.Tick`, or `time.AfterFunc` inside HSM entry, exit, activity, effect, or guard behavior; use `hsm.After`, `hsm.At`, or `hsm.Every`.
- GO-FORB-008.1: External HSM advancement loops such as `switch hsm.TakeSnapshot(ctx, sm).State { ... hsm.Dispatch(ctx, sm, next) }`, `switch sm.State()`, or `strings.Contains(sm.State(), ...)` used to choose follow-on events. Use `hsm.CompletionEventKind` from entry/activity behavior instead.
- GO-FORB-008.2: HSM behavior-dispatched completion/progression events missing `Kind: hsm.CompletionEventKind`, machine-owned error progression events missing `Kind: hsm.ErrorEventKind`, or direct `time.Now()`/random/filesystem/network/environment reads inside HSM behavior.
- GO-FORB-008.3: Conditional dispatch inside HSM behavior that hides transition selection, such as `if condition { hsm.Dispatch(..., completeEvent) }` or `switch payload.Kind { ... hsm.Dispatch(..., nextEvent) }`. Use `hsm.Choice` or guarded transitions.
- GO-FORB-008.4: Behavior-local timers inside HSM entry/exit/activity/effect/guard functions, including `time.After`, `time.Sleep`, `time.NewTimer`, `time.NewTicker`, `time.Tick`, or `time.AfterFunc`. Model time with `hsm.After`, `hsm.At`, or `hsm.Every`.
- GO-11.3: CGO-free concurrency validation. Required CI and quality gates MUST run with `CGO_ENABLED=0`; do not make `-race` a required merge or quality-gate path because the race detector requires CGO.
- CI MUST run `go test ./... -race` for packages that use concurrency (or for the whole project where feasible).
