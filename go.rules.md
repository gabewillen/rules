# Go Coding Rules

## 1. Toolchain & Dependencies
- **No CGO**: Always build with `CGO_ENABLED=0`. CGO, C toolchains, and `import "C"` are strictly forbidden.
- **Minimal Dependencies**: Default to the standard library.
  - Allowed: `stateforward/hsm.go` (state machines) and `agentflare-ai/go-docmd` (docs).
  - Forbidden: External test/assertion frameworks (use built-in `testing`).
  - Require explicit PR justification for any other third-party dependencies.
- **Toolchain Pinning**: Pin exact patch versions for `go` and `toolchain` in `go.mod`. Run `go fix` in a standalone PR when upgrading.
- **Runtime Limits**: Rely on Go 1.25+ container awareness. Do not use `automaxprocs` or override `GOMAXPROCS` unless validated by profiling. Set `GOMEMLIMIT` in containerized environments.
- **Experiments**: Keep `GOEXPERIMENT` empty in production. `goroutineleakprofile` is permitted for CI leak detection.

## 2. Architecture & State Machines (HSM)
- **Mandatory HSM**: Use `stateforward/hsm.go` for multi-step workflows, protocol handlers, lifecycle management, and anything with >2 boolean flags of state. Ad-hoc goroutines and scattered boolean flags are forbidden.
- **State Ownership**: HSM state must reside on the instance struct and be modified exclusively by machine operations.
- **Pure Transitions**: Keep guards pure (no I/O, logging, or non-trivial allocations). Schedule side-effects via operations.
- **Event-Driven**: Dispatch inputs and time events (timeouts, intervals, backoffs) as HSM events.
- **No Blocking**: Never block the HSM event loop on unbounded I/O. Move long I/O to worker goroutines that return events.
- **Naming**: Use hierarchical state names (`Subsystem.State`) and versionable event names (`subsystem.verb`).

## 3. Concurrency
- **Goroutine Lifecycle**: Every goroutine requires an owner, a stop signal (context/closed channel), a join point (`sync.WaitGroup`), and a shutdown deadline. Use `sync.WaitGroup.Go` for simple fan-out. Fire-and-forget goroutines are forbidden.
- **Context Propagation**: Pass `ctx context.Context` as the first parameter. Never store contexts in structs. Apply timeouts at I/O boundaries. Use `context.WithValue` strictly for request-scoped metadata (e.g., trace IDs), never dependency injection.
- **Data Ownership**: Prefer single-writer message passing or sharded ownership over global mutexes. Do not lock across shards.
- **Channels**: Bound all channels and implement backpressure (block with deadline, load-shed, or spill). Select on `ctx.Done()` for blocking sends. Only the sender may close a channel.
- **Locks**: Keep critical sections small and I/O-free. Prefer `sync.Mutex` over `sync.RWMutex` unless profiling justifies it. Use atomics only for simple numeric counters.
- **Rate Limiting**: Use local token buckets or fixed-window counters. Reject requests early (shed load) when saturated.

## 4. Performance & Memory
- **Allocation Minimization**: Avoid `fmt`, `any`, and repeated `[]byte`/`string` conversions in hot paths. Use `io.Reader`/`io.Writer`, `strings.Builder`, `strconv`, and `encoding/hex|base64`.
- **Pre-allocation**: Always pre-size slices and maps (`make([]T, 0, n)`) when bounds are known.
- **Escape Analysis**: Pass small, immutable structs by value. Avoid returning pointers to short-lived locals. Verify escapes with `go test -c -gcflags=all=-m=2`.
- **GC & Lifetimes**: Copy needed data to avoid retaining large backing arrays. Treat `sync.Pool` strictly as a transient cache, not for lifecycle management.
- **Struct Layout**: Group fields from largest to smallest to minimize padding. Separate hot fields from cold fields. Types must be safe when zero-initialized.
- **Timers**: Use `time.NewTimer` and `Reset` carefully. Do not allocate timers in hot loops with `time.After`.

## 5. Testing & Observability
- **Testing Standard**: Write table-driven tests (`t.Run`). Use `testing/synctest` for deterministic concurrent tests—never `time.Sleep`. Fuzz parsers and protocols. CI must run `go test -race`.
- **Benchmarks**: Every hot-path change requires a `testing.B` benchmark reporting allocations (`b.ReportAllocs()`). Isolate setup. Explain any >5% regression.
- **Logging**: Use `log/slog` with a JSON handler in production. Required fields: `component`, `operation`, `error`. Sample or gate debug logs on hot paths.
- **Telemetry**: Expose `net/http/pprof` (behind auth) and collect `runtime/metrics`. Use Go's flight recorder API for traces. OpenTelemetry is restricted to subsystem boundaries with strict cardinality limits.
- **Error Handling**: Wrap context using `%w`. Use `errors.Join` for multiple errors. Define stable sentinel errors for control flow and typed errors for structured data. Never log and return the same error.

## 6. Naming & Layout
- **Project Structure**: `/cmd/<app>` for binaries, `/internal/<subsystem>` for implementation, `/pkg/<name>` for public APIs. No import cycles. No `utils` or `common` dumping grounds.
- **Naming**: Packages must be short, lowercase, and non-stuttering. Exported types use `CamelCase`. Receivers should be 1-2 letters derived from the type (no `this`/`self`).
- **Interfaces**: Keep interfaces small and consumer-defined.
- **Documentation**: Use `doc.go` for package-level docs detailing purpose, concurrency, and performance. Generate READMEs exclusively with `agentflare-ai/go-docmd`. Do not hand-edit generated docs.
