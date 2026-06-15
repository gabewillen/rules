# Dart & Flutter Engineering Rules

## 1. Toolchain & Quality Gates
- **Dart 3.x Only:** Enforce sound null safety (`^3.6.0`+). No opt-outs.
- **Strict Analysis:** Enable `strict-casts`, `strict-inference`, and `strict-raw-types`. Use `package:lints` or `package:flutter_lints`.
- **CI Requirements:** Fail on unformatted code (`dart format -o none --set-exit-if-changed`), analyzer warnings (`--fatal-infos`), and test failures.
- **Pub Management:** Use `dart pub add` and caret (`^`) constraints. Commit `pubspec.lock` for apps, ignore for packages. Support `resolution: workspace` for monorepos. Remove `dependency_overrides` before release.

## 2. Types & Null Safety
- **No `dynamic`:** Ban `dynamic` in public APIs. Use `Object?` and type-narrow, or sealed classes.
- **Typed Collections:** Ban raw `List`/`Map`/`Set`. Always specify generic types.
- **Safe Downcasting:** Ban unchecked `as T`. Use explicit `is` checks or pattern matching.
- **Strict `late`:** Use `late` only for deferred initialization where you guarantee assignment before read.
- **Immutable by Default:** Prefer `final` for locals and fields. Tightly scope any mutable state.
- **Optionals over Sentinels:** Model absent values strictly with `T?`, not magic sentinel values.

## 3. Architecture & API Design
- **Encapsulation:** Treat `lib/src/` as strictly private. Export public APIs solely from `lib/`. Never import another package's `lib/src/`.
- **Class Modifiers:** Use `sealed`, `base`, `interface`, and `final` to explicitly control subclassing and implementation.
- **Single Entrypoint:** Prefer a single primary library export (e.g., `lib/your_pkg.dart`).
- **Doc Comments:** Document all public symbols with `///`, starting with a single-sentence summary.
- **Privacy:** Prefix private file-level or class-level members with an underscore `_`.
- **Deprecations:** Annotate deprecated APIs with `@Deprecated` and a migration hint.

## 4. Error Handling
- **Exceptions vs. Errors:** Throw `Exception` for recoverable failures; `Error` for programmer defects. Never catch `Error` except at top-level process boundaries.
- **Preserve Stack Traces:** Always capture the trace (`catch (e, st)`) and use `rethrow` to bubble up without dropping context.
- **Do Not Swallow Exceptions:** Never fail silently. Add explicit comments if an exception is intentionally ignored.
- **Consistent Boundaries:** Return a typed Result object or throw Exceptions. Do not mix both strategies in the same layer.
- **Top-Level Async Errors:** Wrap server/CLI `main` in `runZonedGuarded` to catch unhandled async errors.

## 5. Async & Concurrency
- **Handle All Futures:** Await, return, or explicitly ignore every `Future` (enforce `unawaited_futures`).
- **Isolate CPU-Bound Work:** Move heavy processing (JSON, images) to Isolates (`Isolate.run()` or `compute()`). Never block the event loop.
- **Resource Cleanup:** Explicitly `cancel()` StreamSubscriptions and `close()` Sinks/Controllers. Do not rely on finalizers for deterministic cleanup.
- **Avoid `async void`:** Return `Future<void>` so callers can await or handle errors, unless constrained by framework event handlers.
- **No Unnecessary Async:** If a function does not `await`, do not mark it `async`. Return the `Future` directly.

## 6. Memory & Performance
- **String Building:** Use `StringBuffer` instead of `+` concatenation inside loops.
- **Const Instantiation:** Maximize use of `const` constructors and literals to reduce allocations.
- **Preallocate Collections:** Use `List.filled` or `List.generate` when the final size is known.
- **Cache Expensive Objects:** Avoid per-call allocations for heavy objects like `RegExp` or `DateFormat`. Cache them in static/final fields.

## 7. Data & Serialization
- **Zero-Trust Boundaries:** Validate all external input (JSON, network, platform channels) immediately at the edge. Never log sensitive data or hardcode secrets.
- **Typed Deserialization:** Convert dynamic JSON maps into strongly-typed domain models using `dart:convert`. Never leak `Map<String, dynamic>` into core business logic.
- **Code Generation:** Use schema-driven generation (e.g., `json_serializable`) for complex or long-lived API models.

## 8. Interop & Platform Channels
- **Adapter Boundaries:** Isolate FFI, JS Interop, and Platform Channels behind strict, testable adapter layers.
- **No Leaky Abstractions:** Never expose native/JS types in Dart APIs.
- **Modern Web/Native Tooling:** Use `package:web` for JS interop and `ffigen` for native bindings. Avoid handwritten FFI at scale.
- **Type-Safe Channels:** Use versioned, typed messages for Flutter platform channels. Prefer Pigeon over untyped standard method channels.

## 9. Testing
- **Determinism:** Fake/mock external I/O (network, filesystem, databases). Avoid flaky time-based assertions; use fake async or controlled clocks.
- **Right-Sizing:** Map unit tests to pure logic, widget tests to UI components, and integration tests to complete user flows.
- **Regression Guarding:** Every bug fix must include a targeted regression test.
