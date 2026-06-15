---
alwaysApply: true
---
# Dart 2025-2026 Opinionated Governance Ruleset (Dart 3.x)
This document is a **rulebook for AI coding agents** writing and maintaining Dart (including Flutter and server/CLI). It is **not** a tutorial.
Each rule has an ID (**R###**) and uses normative language (**MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, **MAY**). Rules are intended to be enforceable by code review, static analysis, or CI.
---
## 1. Scope and assumptions
- **R001** MUST write code that compiles and runs under **sound null safety**; codebases MUST NOT opt out of null safety. [S25]
- **R002** MUST assume Dart **3.x** language semantics and tooling; features that require a minimum language version MUST NOT be used unless the package SDK constraint is compatible. [S12]
- **R003** MUST treat Dart files as libraries and rely on Dart library privacy (leading underscore) for internal encapsulation. [S61]
- **R004** MUST keep this ruleset enforceable via automated tooling: `dart format`, `dart analyze`, and `dart test` (or `flutter test`) MUST be runnable non-interactively in CI. [S6][S7][S23]
- **R005** SHOULD default to cross-platform Dart where feasible; platform-specific code MUST be isolated behind explicit interop boundaries (see Interop rules). [S52][S48][S54]

## 2. Toolchain and versions (Dart SDK, pub, analyzer)
Command expectations (CI friendly):
```sh
# Inspect toolchain versions
dart --version
flutter --version # only in Flutter repos
# Verify the tool exists and is usable
dart --help
```
- **R006** MUST declare an SDK constraint in every `pubspec.yaml` (`environment: sdk:`) and keep it accurate for the oldest supported Dart version. [S19]
- **R007** SHOULD run CI on the latest stable Dart SDK (Dart 3.11 as of February 2026) and SHOULD also run on the oldest supported SDK if you publish a package. [S12][S13]
- **R008** MUST use `dart` as the source of truth for formatting, analysis, and tests (`dart format`, `dart analyze`, `dart test`). Flutter projects MAY also run `flutter analyze`/`flutter test` in addition. [S6][S7][S23]
- **R009** MUST NOT rely on deprecated lint bundles (for example, `pedantic`); MUST use Dart team recommended lint sets via `package:lints` or `package:flutter_lints`. [S8][S9][S10]
- **R010** MAY use analyzer plugins for organization-specific enforceable rules; if used, the minimum SDK MUST be compatible with analyzer plugin support (introduced in Dart 3.10). [S11][S12]
- **R011** MAY use pub workspaces for monorepos; if used, all workspace packages MUST declare SDK constraints of `^3.6.0` or higher and use `resolution: workspace`. [S20]

## 3. Formatting and style (dart format, naming, imports, doc comments)
Commands (format as a gate):
```sh
# Format the whole package
dart format .
# CI gate: fail if formatting would change files
dart format -o none --set-exit-if-changed .
```
[S6]
- **R012** MUST format all Dart source with `dart format`; code reviews MUST reject manual formatting conventions that conflict with formatter output. [S2][S6]
- **R013** MUST gate merges on `dart format -o none --set-exit-if-changed` (or an equivalent check) to prevent unformatted commits. [S6]
- **R014** MUST follow Effective Dart identifier naming conventions (e.g., lowerCamelCase for members, UpperCamelCase for types, and leading underscore for private). [S2][S5]
- **R015** MUST NOT use relative imports that escape `lib/` (for example `../../`); public library code MUST import via `package:your_pkg/...` to keep imports stable. [S4][S61]
- **R016** MUST NOT import another package’s `lib/src/...` symbols; `lib/src` is private implementation. [S22]
- **R017** SHOULD keep import directives ordered and grouped consistently (Dart SDK imports, then package imports, then relative imports) and SHOULD avoid unused imports (enforced by analyzer). [S4][S7]
- **R018** MUST write doc comments (`///`) for every public library, public class, public enum, public extension type, and every public top-level or member that is not self-explanatory. [S3]
- **R019** MUST start doc comments with a single sentence summary that ends with a period. [S3]
- **R020** MUST NOT write doc comments that restate the identifier name without adding meaning. [S3]

## 4. Static analysis and linting (analysis_options, recommended lints, when to suppress)
Example `analysis_options.yaml` (baseline, non-Flutter):
```yaml
include: package:lints/recommended.yaml
analyzer:
  exclude:
    - build/**
    - **/*.g.dart
    - **/*.freezed.dart
  language:
    strict-casts: true
    strict-inference: true
    strict-raw-types: true
linter:
  rules:
    # Lifecycle safety
    - cancel_subscriptions
    - close_sinks
    # Async safety
    - unawaited_futures
    - discarded_futures
    # Performance footguns
    - use_string_buffers
```
[S8][S9][S34][S35][S36]
Example `analysis_options.yaml` (Flutter package/app):
```yaml
include: package:flutter_lints/recommended.yaml
analyzer:
  exclude:
    - build/**
    - **/*.g.dart
  language:
    strict-casts: true
    strict-inference: true
    strict-raw-types: true
```
[S8][S10]
- **R021** MUST have an `analysis_options.yaml` at each package root (next to `pubspec.yaml`). [S8]
- **R022** MUST include Dart team recommended lints (`package:lints`) for pure Dart packages, and MUST include `package:flutter_lints` for Flutter packages. [S8][S9][S10]
- **R023** MUST enable `strict-casts`, `strict-inference`, and `strict-raw-types` under `analyzer: language:` unless the project has a documented reason not to. [S8]
- **R024** MUST run `dart analyze` in CI and MUST fail CI on errors and warnings; CI SHOULD also use `--fatal-infos`. [S7]
- **R025** MUST configure analyzer excludes for generated output and build artifacts (at minimum `build/**` and generated `.g.dart` style files). [S8]
- **R026** MUST NOT suppress diagnostics globally (for example by disabling large lint sets) to "make CI green"; suppressions MUST be narrow and justified.
- **R027** If `// ignore:` or `// ignore_for_file:` is used, it MUST include only the minimal set of codes and MUST be accompanied by a short reason comment on the same line or immediately above. [S8]
- **R028** MAY raise severities (warnings to errors) for project-critical diagnostics via `analyzer: errors:`; if you do, you MUST document the rationale in the same file. [S8]
- **R029** MAY use analyzer plugins to enforce architectural rules (import boundaries, banned APIs, etc.); if used, they MUST run in CI via `dart analyze`. [S11][S7]

## 5. Types and null safety (strictness, late, required, casts, generics)
- **R030** MUST keep the codebase on sound null safety; nullable types (`T?`) MUST be used intentionally, not as a convenience. [S25]
- **R031** MUST enable strict type checks (`strict-casts`, `strict-inference`, `strict-raw-types`) and treat violations as defects, not style issues. [S8]
- **R032** MUST NOT use `dynamic` in public APIs. If a value is "unknown", use `Object?` plus narrowing, or introduce a sealed domain type. [S26][S55]
- **R033** MUST NOT use `late` for convenience. `late` MAY be used only when (a) initialization cannot be done at declaration, AND (b) you can prove the field is set before any read. [S25]
- **R034** MUST prefer `final` for locals and fields unless mutation is required for correctness; mutable state MUST be tightly scoped.
- **R035** MUST type all collections that cross a boundary (public APIs, JSON, isolates, FFI). Raw `List` / `Map` / `Set` declarations MUST NOT appear. [S8]
- **R036** MUST avoid unchecked downcasts (`as T`) unless immediately preceded by a type check, pattern match, or explicit validation; casts MUST be local and commented when non-obvious. [S26]
- **R037** SHOULD use generics to keep APIs type-safe; `Object?` and `dynamic` SHOULD NOT be used where a type parameter is feasible. [S26]
- **R038** SHOULD model "value may be absent" using `T?` (or an explicit `Option`-like type) rather than sentinel values.

## 6. API design (public vs internal, library boundaries, stability, deprecations)
- **R039** Public API MUST live under `lib/` and be reachable via `package:your_pkg/...` imports; anything under `lib/src/` MUST be treated as private implementation. [S22]
- **R040** MUST NOT require consumers to import `lib/src/...`; if internal types need to be exposed, they MUST be re-exported from a `lib/*.dart` entrypoint. [S22]
- **R041** MUST keep the public surface small: prefer a single primary entrypoint (`lib/your_pkg.dart`) that exports the intended API, and keep implementation details in `lib/src`. [S22]
- **R042** MUST document stability expectations for every public library: stable, experimental, or internal-only. (This is a doc comment requirement.) [S3]
- **R043** MUST follow Effective Dart design guidelines for API consistency (naming, parameter ordering, and user-centric documentation). [S5][S3]
- **R044** SHOULD use class modifiers (`sealed`, `base`, `interface`, `final`) to protect package APIs from unintended extension/implementation when maintaining a library. [S55][S56]
- **R045** If an API is intended for extension by downstream packages, it MUST be explicitly designed for it (documented invariants, protected hooks, and tests). Otherwise, it SHOULD be `final` or `base`. [S56]
- **R046** MUST use semantic versioning for published packages; breaking changes MUST be released as a new major version and MUST be called out in CHANGELOG.
- **R047** If deprecating an API, MUST annotate it with `@Deprecated` including a migration hint, and MUST keep the deprecated API functional for at least one stable release cycle unless it is a security issue.

## 7. Error handling (exceptions vs Result patterns, stack traces, logging guidance)
- **R048** MUST throw only `Exception` or `Error` subtypes (or custom types implementing `Exception`); MUST NOT throw strings, numbers, or other arbitrary objects. [S27]
- **R049** MUST treat `Error` as a programmer defect: code MUST NOT catch `Error` except at process boundaries (top-level isolate, test harness) for logging and crash reporting. [S27]
- **R050** When catching exceptions, MUST capture the stack trace (`catch (e, st)`) and MUST preserve it when rethrowing or wrapping; use `rethrow` when you are not transforming the error.
- **R051** MUST NOT swallow exceptions. If an exception is intentionally ignored, code MUST include a comment explaining why it is safe.
- **R052** At each package boundary, MUST choose one error transport style for the public API: (A) throw exceptions, or (B) return an explicit result type. Mixing both styles in the same layer MUST NOT occur.
- **R053** If using a Result pattern, failures MUST be typed (no stringly-typed error codes) and MUST include the original exception and stack trace when wrapping.
- **R054** Process entrypoints (`main`) in server/CLI code SHOULD run inside `runZonedGuarded` (or equivalent) to report uncaught async errors consistently. [S30]
- **R055** Logging at catch sites MUST include enough context to diagnose the failure (operation, identifiers, and error + stack trace) and MUST NOT include secrets. [S47]

## 8. Async (Futures, Streams, cancellation, timeouts, backpressure guidance)
- **R056** MUST enable and pass the `unawaited_futures` and `discarded_futures` lints; every created `Future` MUST be awaited, returned, or intentionally ignored with an explicit justification. [S28]
- **R057** MUST NOT block the event loop with synchronous heavy work inside async code; CPU-heavy work MUST move to an isolate (see Concurrency rules). [S31][S32]
- **R058** MUST use `async`/`await` for readability over chaining `then()` in application code, except in low-level libraries where allocation/micro-optimizations are proven necessary. [S28]
- **R059** MUST propagate cancellation for `StreamSubscription`s by calling `cancel()` in the owning lifecycle (`dispose`, `close`, or `tearDown`). [S29][S34]
- **R060** MUST close any `StreamController`, `Sink`, or `IOSink` that your code creates or owns. [S35]
- **R061** MUST set explicit timeouts for network and external I/O operations; default timeouts MUST be documented and configurable.
- **R062** SHOULD model cancellation for non-stream async operations using `package:async` `CancelableOperation` or an explicit cancellation token pattern. [S41]
- **R063** If using `CancelableOperation`, MUST provide an `onCancel` behavior (or equivalent) if the underlying work is expensive; canceling without stopping work MUST be treated as a bug unless the cost is negligible. [S41]
- **R064** For streams that may outpace consumers, MUST define backpressure strategy: (A) pause/resume via `StreamSubscription`, (B) buffering with explicit bounds, or (C) dropping/coalescing with documented semantics. [S29]
- **R065** MUST handle stream errors explicitly (via `onError` or `try/catch` around `await for`) and MUST avoid silently terminating stream processing on first error unless that is the specified behavior. [S29]

## 9. Concurrency and isolates (when to use isolates, message passing patterns, pitfalls)
- **R066** MUST understand that concurrency in Dart includes async APIs (`Future`/`Stream`) and isolates; isolates are the only way to run Dart code in parallel on multiple cores. [S31]
- **R067** MUST use isolates when computations are large enough to block other work (especially Flutter UI), such as large JSON parsing, image/audio/video processing, or heavy filtering. [S32]
- **R068** MUST NOT attempt shared-memory concurrency between isolates; isolates have isolated memory and communicate by message passing only. [S31][S33]
- **R069** For one-shot work, SHOULD prefer `Isolate.run()` (or Flutter `compute`) over manually managing `ReceivePort`/`SendPort`, unless you need long-lived worker isolates. [S32]
- **R070** If you create long-lived isolates, MUST implement an explicit lifecycle protocol: start, ready, request, response, and shutdown messages; worker isolates MUST close ports on shutdown.
- **R071** MUST keep isolate messages versioned and typed: define request/response DTOs (simple, serializable shapes) and avoid ad-hoc `Map<String, dynamic>` payloads.
- **R072** MUST treat isolate boundaries as failure boundaries: worker failures MUST be surfaced to the main isolate with enough context, and MUST not be silently dropped. [S32]
- **R073** MUST consider platform constraints: Flutter web does not support multiple isolates; code that depends on additional isolates MUST provide a single-isolate fallback or be gated by platform. [S32]

## 10. Performance (allocation, collections, string building, sync vs async overhead)
- **R074** MUST avoid string concatenation in loops; MUST use `StringBuffer` (or equivalent) when building strings iteratively. [S36][S37]
- **R075** MUST NOT mark a function `async` if it does not `await`; return the `Future` directly to avoid extra scheduling and allocation.
- **R076** SHOULD prefer `const` constructors and `const` literals when values are compile-time constants, to reduce allocations and improve canonicalization.
- **R077** SHOULD avoid per-call allocations in hot paths (for example, repeatedly allocating `RegExp`, `DateFormat`, or large temporary collections); move reusable objects to `final` top-level or cached singletons.
- **R078** If collection size is known, SHOULD preallocate (for example, `List.filled` or `List.generate`) instead of repeated growth in a loop.
- **R079** MUST justify any micro-optimization that reduces readability with a benchmark or profiler evidence (see Benchmark rules). [S42]

## 11. Memory and lifecycle (resource cleanup, close/dispose patterns, finalizers cautions)
- **R080** MUST treat resource ownership explicitly: the code that creates a resource is responsible for closing/disposing it, unless ownership is transferred and documented.
- **R081** MUST close `Sink`s (including `IOSink`) that your code creates; leaving them open is a defect and is lintable. [S35]
- **R082** MUST cancel `StreamSubscription`s that your code creates; leaving them active is a defect and is lintable. [S34]
- **R083** MUST use `try`/`finally` (or `using`-style helper) around resources that must be released even on error (files, sockets, subscriptions, ports).
- **R084** MUST NOT rely on garbage collection for releasing external resources; explicit `close()`/`dispose()` is required.
- **R085** MAY use `Finalizer`/`NativeFinalizer` only as a last-resort safety net for external resources; Finalizers are not guaranteed to run and MUST NOT be the primary cleanup path. [S39][S40]
- **R086** When investigating leaks or excessive allocations, SHOULD use DevTools memory tooling and MUST attach evidence to the change that claims to fix a leak. [S38]

## 12. Data and serialization (json, custom codecs, validation, schema discipline)
- **R087** MUST treat all external input (JSON, query params, headers, files, platform messages) as untrusted and validate it at the boundary before use. [S47]
- **R088** MUST use `dart:convert` (`jsonDecode`/`jsonEncode`) for JSON unless there is a documented reason to use an alternate codec. [S44][S43]
- **R089** MUST NOT let `dynamic` JSON maps leak past the boundary: decode, validate, and convert to typed domain models immediately. [S45]
- **R090** SHOULD implement typed `fromJson` (or equivalent) constructors/factories for models and keep them next to the model type; parsing logic MUST be unit-tested. [S45][S24]
- **R091** SHOULD use pattern matching or structured checks to validate JSON shape (keys, types, optionality) rather than assuming the shape. [S45]
- **R092** For large model graphs or long-lived APIs, SHOULD use code generation (for example `json_serializable`) or an equivalent schema-driven approach to avoid handwritten parsing drift. [S43]
- **R093** MUST version serialized formats that cross service boundaries; backwards compatibility rules MUST be documented (added fields optional, removed fields supported for at least one release).
- **R094** Custom codecs MUST be deterministic and MUST reject invalid inputs with typed errors that preserve context (field path, reason).

## 13. Package layout and pubspec practices (versioning, dependency hygiene, overrides)
Common pub workflows:
```sh
# Add a dependency (writes pubspec.yaml and runs resolution)
dart pub add http
# Add a dev dependency
dart pub add dev:test
# Resolve dependencies after editing pubspec.yaml
dart pub get
# Review outdated packages and recommended upgrades
dart pub outdated
# Apply upgrades within your constraints (apps: review pubspec.lock diffs)
dart pub upgrade
```
[S14][S15][S16][S17]
- **R095** MUST follow pub package layout conventions (top-level `lib/`, `test/`, `bin/`, `example/`, etc.) unless there is a documented reason not to. [S21]
- **R096** MUST keep implementation code under `lib/src/` and keep `lib/` focused on the public API surface. [S22]
- **R097** For application packages (runnable apps), MUST commit `pubspec.lock` to source control. [S62][S15]
- **R098** For library packages intended to be depended on by others, MUST NOT commit `pubspec.lock` (to allow a range of dependency resolutions). [S15][S62]
- **R099** MUST use `dart pub add` to add dependencies (and `dart pub add dev:` for dev dependencies) instead of manual edits, unless automation requires otherwise. [S14]
- **R100** MUST run `dart pub get` after any `pubspec.yaml` change and MUST keep dependency resolution reproducible. [S15][S19]
- **R101** SHOULD use caret constraints (`^`) for most dependencies and SHOULD avoid overly tight pins unless required for correctness or security. [S14][S18]
- **R102** MUST review and minimize `dependency_overrides`; overrides MUST NOT ship in published packages and MUST be removed before release. [S18]
- **R103** If using workspaces, SHOULD keep `dependency_overrides` centralized (preferably in the root `pubspec.yaml`) and MUST ensure each overridden package is overridden at most once. [S20][S18]
- **R104** MUST use `dart pub outdated` regularly and MUST update dependencies intentionally with tests. [S16]
- **R105** MUST run `dart pub upgrade` intentionally (not automatically in every CI run) and MUST review lockfile diffs for applications. [S17][S16]
- **R106** For packages published to pub.dev, MUST maintain a CHANGELOG and MUST use `dart pub publish` (or automated publishing) with tags/releases. [S59][S58]

## 14. Testing (unit/widget/integration where relevant, determinism, fakes/mocks)
- **R107** MUST have automated tests and MUST be runnable via `dart test` for pure Dart packages and via `flutter test` for Flutter packages. [S23]
- **R108** MUST place Dart unit tests under `test/` and keep tests deterministic (no real network, no real time, no real randomness without seeding). [S23][S24]
- **R109** Flutter codebases MUST use the appropriate test level: unit tests for pure logic, widget tests for widgets, and integration tests for full app flows. [S64]
- **R110** MUST isolate external dependencies in tests (filesystem, network, databases, platform channels) using fakes or mocks; tests MUST NOT require external services to be available. [S64]
- **R111** MUST ensure every bug fix includes (or updates) a regression test that fails before the fix and passes after it.
- **R112** SHOULD prefer fakes/stubs over deep mocking when behavior is complex; mocks MUST be limited to interface boundaries and MUST not assert internal implementation details.
- **R113** MUST avoid flaky time-based assertions; prefer controlled clocks, fake async, or explicit synchronization points.
- **R114** Integration tests MUST be scoped to critical flows and MUST run on CI for at least one representative target environment when shipping apps. [S64]

## 15. CI and quality gates (format, analyze, test, coverage policy suggestions)
Minimum CI commands (per package):
```sh
# Resolve dependencies
dart pub get
# Enforce formatting
dart format -o none --set-exit-if-changed .
# Enforce analysis (treat infos as fatal)
dart analyze --fatal-infos
# Run tests
dart test
```
[S6][S7][S15][S23]
- **R115** CI MUST run `dart pub get`, `dart format -o none --set-exit-if-changed`, `dart analyze --fatal-infos`, and `dart test` (or `flutter test`) on every change. [S6][S7][S15][S23]
- **R116** CI MUST fail on any formatting change, analysis warning, or test failure; "warnings allowed" policies MUST NOT be used. [S6][S7]
- **R117** CI SHOULD cache the pub cache and build artifacts appropriately, but MUST NOT hide dependency resolution or advisory warnings. [S46]
- **R118** Projects SHOULD track coverage and set a ratcheting minimum (coverage must not decrease) rather than a fixed global threshold.
- **R119** Monorepos using pub workspaces MUST run CI from the workspace root so analysis and dependency resolution are unified. [S20]
- **R120** Release pipelines for published packages SHOULD use automated publishing with tokens stored in CI secret storage, never in the repo. [S58][S59]

## 16. Security and safety basics (secrets, inputs, network, deserialization risks)
- **R121** MUST NOT hardcode secrets (API keys, tokens, private endpoints) in source, tests, or build scripts; secrets MUST come from a secure runtime configuration or secret manager. [S47]
- **R122** MUST treat all external inputs as hostile and validate early; validation failures MUST be handled as controlled errors, not crashes. [S47][S45]
- **R123** MUST prefer secure transport (TLS/HTTPS) for network traffic; if non-TLS is required (rare), it MUST be explicitly justified and isolated.
- **R124** MUST monitor and respond to dependency security advisories surfaced by the pub client during `dart pub get`. [S46]
- **R125** If suppressing an advisory using `ignored_advisories`, MUST document why it is not relevant and MUST revisit the suppression at each dependency update. [S46]
- **R126** MUST avoid logging sensitive data (tokens, auth headers, full payloads with personal data). [S47]
- **R127** MUST keep `dependency_overrides` out of releases and published artifacts; overrides can hide vulnerable transitive versions. [S18]

## 17. Interop (FFI, JS interop, platform considerations) as rules, not tutorials
- **R128** Interop code MUST be isolated behind a small, testable boundary (adapter layer); application layers MUST depend on Dart domain interfaces, not platform-specific APIs. [S54][S52][S48]
- **R129** FFI usage MUST be limited to Dart Native platforms; code that must run on web MUST NOT import `dart:ffi`. [S52]
- **R130** FFI bindings MUST be generated where possible (for example with `package:ffigen` for Apple interop) and MUST NOT be handwritten at scale. [S53]
- **R131** FFI code MUST manage native memory explicitly (allocate and deallocate correctly) and MUST document ownership for every pointer returned or stored. [S52]
- **R132** JS interop MUST use the modern interop stack (`dart:js_interop` and related guidance) and SHOULD avoid legacy interop APIs except when maintaining existing code. [S48][S49][S51]
- **R133** Web platform API access SHOULD migrate to `package:web` where applicable; new code MUST NOT start on deprecated web interop surfaces. [S50][S49]
- **R134** Interop surfaces MUST NOT leak JS/FFI/platform-channel types into public package APIs; wrap them into domain types or extension types that you own. [S57][S48]
- **R135** Flutter platform channel messages MUST be versioned and typed; where possible, SHOULD use Pigeon-generated, type-safe channels instead of untyped maps. [S54]
- **R136** Platform channel boundaries MUST validate inputs and MUST handle platform exceptions without crashing the Dart isolate. [S54][S27]

## 18. Anti-patterns list (explicit “do not do this” items)
- **R137** MUST NOT commit `.dart_tool/`, `build/`, or generated `doc/api/` outputs to source control. [S62]
- **R138** MUST NOT commit `pubspec.lock` for library packages; MUST NOT delete it for application packages. [S62]
- **R139** MUST NOT import from another package’s `lib/src/...`. [S22]
- **R140** MUST NOT use `dynamic` (or `Map<String, dynamic>`) as an architectural shortcut across layers (including JSON and isolate messaging). [S45]
- **R141** MUST NOT start background Futures without handling errors; unhandled async errors MUST be treated as defects. [S30][S28]
- **R142** MUST NOT use `async void` except for UI/event handler entrypoints where the framework requires it; prefer `Future<void>` so failures can be observed.
- **R143** MUST NOT use `dependency_overrides` as a long-term solution; overrides MUST be temporary and tracked. [S18]
- **R144** MUST NOT ignore pub security advisories without documenting why they are irrelevant. [S46]
- **R145** MUST NOT rely on finalizers as deterministic cleanup. [S39]
- **R146** MUST NOT use platform channels or JS interop with untyped `Map` payloads as the public boundary; define typed messages and versioning. [S54][S48]
- **R147** MUST NOT move CPU-bound work onto the main isolate in Flutter apps in a way that can jank the UI; use isolates or platform-native optimizations. [S32][S33]


## Additional Merged Rules

- R001: MUST write code that compiles and runs under **sound null safety**; projects MUST NOT opt out of null safety. [S25]
- R002: MUST assume Dart **3.x** language semantics and tooling; features that require a minimum language version MUST NOT be used unless the package SDK constraint is compatible. [S12]
- R003: MUST treat Dart files as libraries and rely on Dart library privacy (leading underscore) for internal encapsulation. [S61]
- R004: MUST keep this ruleset enforceable via automated tooling: `dart format`, `dart analyze`, and `dart test` (or `flutter test`) MUST be runnable non-interactively in CI. [S6][S7][S23]
- R005: SHOULD default to cross-platform Dart where feasible; platform-specific code MUST be isolated behind explicit interop boundaries (see Interop rules). [S52][S48][S54]
- R006: MUST declare an SDK constraint in every `pubspec.yaml` (`environment: sdk:`) and keep it accurate for the oldest supported Dart version. [S19]
- R007: SHOULD run CI on the latest stable Dart SDK (Dart 3.11 as of February 2026) and SHOULD also run on the oldest supported SDK if you publish a package. [S12][S13]
- R008: MUST use `dart` as the source of truth for formatting, analysis, and tests (`dart format`, `dart analyze`, `dart test`). Flutter projects MAY also run `flutter analyze`/`flutter test` in addition. [S6][S7][S23]
- R009: MUST NOT rely on deprecated lint bundles (for example, `pedantic`); MUST use Dart team recommended lint sets via `package:lints` or `package:flutter_lints`. [S8][S9][S10]
- R010: MAY use analyzer plugins for organization-specific enforceable rules; if used, the minimum SDK MUST be compatible with analyzer plugin support (introduced in Dart 3.10). [S11][S12]
- R011: MAY use pub workspaces for monorepos; if used, all workspace packages MUST declare SDK constraints of `^3.6.0` or higher and use `resolution: workspace`. [S20]
- R012: MUST format all Dart source with `dart format`; code reviews MUST reject manual formatting conventions that conflict with formatter output. [S2][S6]
- R013: MUST gate merges on `dart format -o none --set-exit-if-changed` (or an equivalent check) to prevent unformatted commits. [S6]
- R014: MUST follow Effective Dart identifier naming conventions (e.g., lowerCamelCase for members, UpperCamelCase for types, and leading underscore for private). [S2][S5]
- R015: MUST NOT use relative imports that escape `lib/` (for example `../../`); public library code MUST import via `package:your_pkg/...` to keep imports stable. [S4][S61]
- R016: MUST NOT import another package’s `lib/src/...` symbols; `lib/src` is private implementation. [S22]
- R017: SHOULD keep import directives ordered and grouped consistently (Dart SDK imports, then package imports, then relative imports) and SHOULD avoid unused imports (enforced by analyzer). [S4][S7]
- R018: MUST write doc comments (`///`) for every public library, public class, public enum, public extension type, and every public top-level or member that is not self-explanatory. [S3]
- R019: MUST start doc comments with a single sentence summary that ends with a period. [S3]
- R020: MUST NOT write doc comments that restate the identifier name without adding meaning. [S3]
- R021: MUST have an `analysis_options.yaml` at each package root (next to `pubspec.yaml`). [S8]
- R022: MUST include Dart team recommended lints (`package:lints`) for pure Dart packages, and MUST include `package:flutter_lints` for Flutter packages. [S8][S9][S10]
- R023: MUST enable `strict-casts`, `strict-inference`, and `strict-raw-types` under `analyzer: language:` unless the project has a documented reason not to. [S8]
- R024: MUST run `dart analyze` in CI and MUST fail CI on errors and warnings; CI SHOULD also use `--fatal-infos`. [S7]
- R025: MUST configure analyzer excludes for generated output and build artifacts (at minimum `build/**` and generated `.g.dart` style files). [S8]
- R026: MUST NOT suppress diagnostics globally (for example by disabling large lint sets) to "make CI green"; suppressions MUST be narrow and justified.
- R027: If `// ignore:` or `// ignore_for_file:` is used, it MUST include only the minimal set of codes and MUST be accompanied by a short reason comment on the same line or immediately above. [S8]
- R028: MAY raise severities (warnings to errors) for project-critical diagnostics via `analyzer: errors:`; if you do, you MUST document the rationale in the same file. [S8]
- R029: MAY use analyzer plugins to enforce architectural rules (import boundaries, banned APIs, etc.); if used, they MUST run in CI via `dart analyze`. [S11][S7]
- R030: MUST keep the project on sound null safety; nullable types (`T?`) MUST be used intentionally, not as a convenience. [S25]
- R031: MUST enable strict type checks (`strict-casts`, `strict-inference`, `strict-raw-types`) and treat violations as defects, not style issues. [S8]
- R032: MUST NOT use `dynamic` in public APIs. If a value is "unknown", use `Object?` plus narrowing, or introduce a sealed domain type. [S26][S55]
- R033: MUST NOT use `late` for convenience. `late` MAY be used only when (a) initialization cannot be done at declaration, AND (b) you can prove the field is set before any read. [S25]
- R034: MUST prefer `final` for locals and fields unless mutation is required for correctness; mutable state MUST be tightly scoped.
- R035: MUST type all collections that cross a boundary (public APIs, JSON, isolates, FFI). Raw `List` / `Map` / `Set` declarations MUST NOT appear. [S8]
- R036: MUST avoid unchecked downcasts (`as T`) unless immediately preceded by a type check, pattern match, or explicit validation; casts MUST be local and commented when non-obvious. [S26]
- R037: SHOULD use generics to keep APIs type-safe; `Object?` and `dynamic` SHOULD NOT be used where a type parameter is feasible. [S26]
- R038: SHOULD model "value may be absent" using `T?` (or an explicit `Option`-like type) rather than sentinel values.
- R039: Public API MUST live under `lib/` and be reachable via `package:your_pkg/...` imports; anything under `lib/src/` MUST be treated as private implementation. [S22]
- R040: MUST NOT require consumers to import `lib/src/...`; if internal types need to be exposed, they MUST be re-exported from a `lib/*.dart` entrypoint. [S22]
- R041: MUST keep the public surface small: prefer a single primary entrypoint (`lib/your_pkg.dart`) that exports the intended API, and keep implementation details in `lib/src`. [S22]
- R042: MUST document stability expectations for every public library: stable, experimental, or internal-only. (This is a doc comment requirement.) [S3]
- R043: MUST follow Effective Dart design guidelines for API consistency (naming, parameter ordering, and user-centric documentation). [S5][S3]
- R044: SHOULD use class modifiers (`sealed`, `base`, `interface`, `final`) to protect package APIs from unintended extension/implementation when maintaining a library. [S55][S56]
- R045: If an API is intended for extension by downstream packages, it MUST be explicitly designed for it (documented invariants, protected hooks, and tests). Otherwise, it SHOULD be `final` or `base`. [S56]
- R046: MUST use semantic versioning for published packages; breaking changes MUST be released as a new major version and MUST be called out in CHANGELOG.
- R047: If deprecating an API, MUST annotate it with `@Deprecated` including a migration hint, and MUST keep the deprecated API functional for at least one stable release cycle unless it is a security issue.
- R048: MUST throw only `Exception` or `Error` subtypes (or custom types implementing `Exception`); MUST NOT throw strings, numbers, or other arbitrary objects. [S27]
- R049: MUST treat `Error` as a programmer defect: code MUST NOT catch `Error` except at process boundaries (top-level isolate, test harness) for logging and crash reporting. [S27]
- R050: When catching exceptions, MUST capture the stack trace (`catch (e, st)`) and MUST preserve it when rethrowing or wrapping; use `rethrow` when you are not transforming the error.
- R051: MUST NOT swallow exceptions. If an exception is intentionally ignored, code MUST include a comment explaining why it is safe.
- R052: At each package boundary, MUST choose one error transport style for the public API: (A) throw exceptions, or (B) return an explicit result type. Mixing both styles in the same layer MUST NOT occur.
- R053: If using a Result pattern, failures MUST be typed (no stringly-typed error codes) and MUST include the original exception and stack trace when wrapping.
- R054: Process entrypoints (`main`) in server/CLI code SHOULD run inside `runZonedGuarded` (or equivalent) to report uncaught async errors consistently. [S30]
- R055: Logging at catch sites MUST include enough context to diagnose the failure (operation, identifiers, and error + stack trace) and MUST NOT include secrets. [S47]
- R056: MUST enable and pass the `unawaited_futures` and `discarded_futures` lints; every created `Future` MUST be awaited, returned, or intentionally ignored with an explicit justification. [S28]
- R057: MUST NOT block the event loop with synchronous heavy work inside async code; CPU-heavy work MUST move to an isolate (see Concurrency rules). [S31][S32]
- R058: MUST use `async`/`await` for readability over chaining `then()` in application code, except in low-level libraries where allocation/micro-optimizations are proven necessary. [S28]
- R059: MUST propagate cancellation for `StreamSubscription`s by calling `cancel()` in the owning lifecycle (`dispose`, `close`, or `tearDown`). [S29][S34]
- R060: MUST close any `StreamController`, `Sink`, or `IOSink` that your code creates or owns. [S35]
- R061: MUST set explicit timeouts for network and external I/O operations; default timeouts MUST be documented and configurable.
- R062: SHOULD model cancellation for non-stream async operations using `package:async` `CancelableOperation` or an explicit cancellation token pattern. [S41]
- R063: If using `CancelableOperation`, MUST provide an `onCancel` behavior (or equivalent) if the underlying work is expensive; canceling without stopping work MUST be treated as a bug unless the cost is negligible. [S41]
- R064: For streams that may outpace consumers, MUST define backpressure strategy: (A) pause/resume via `StreamSubscription`, (B) buffering with explicit bounds, or (C) dropping/coalescing with documented semantics. [S29]
- R065: MUST handle stream errors explicitly (via `onError` or `try/catch` around `await for`) and MUST avoid silently terminating stream processing on first error unless that is the specified behavior. [S29]
- R066: MUST understand that concurrency in Dart includes async APIs (`Future`/`Stream`) and isolates; isolates are the only way to run Dart code in parallel on multiple cores. [S31]
- R067: MUST use isolates when computations are large enough to block other work (especially Flutter UI), such as large JSON parsing, image/audio/video processing, or heavy filtering. [S32]
- R068: MUST NOT attempt shared-memory concurrency between isolates; isolates have isolated memory and communicate by message passing only. [S31][S33]
- R069: For one-shot work, SHOULD prefer `Isolate.run()` (or Flutter `compute`) over manually managing `ReceivePort`/`SendPort`, unless you need long-lived worker isolates. [S32]
- R070: If you create long-lived isolates, MUST implement an explicit lifecycle protocol: start, ready, request, response, and shutdown messages; worker isolates MUST close ports on shutdown.
- R071: MUST keep isolate messages versioned and typed: define request/response DTOs (simple, serializable shapes) and avoid ad-hoc `Map<String, dynamic>` payloads.
- R072: MUST treat isolate boundaries as failure boundaries: worker failures MUST be surfaced to the main isolate with enough context, and MUST not be silently dropped. [S32]
- R073: MUST consider platform constraints: Flutter web does not support multiple isolates; code that depends on additional isolates MUST provide a single-isolate fallback or be gated by platform. [S32]
- R074: MUST avoid string concatenation in loops; MUST use `StringBuffer` (or equivalent) when building strings iteratively. [S36][S37]
- R075: MUST NOT mark a function `async` if it does not `await`; return the `Future` directly to avoid extra scheduling and allocation.
- R076: SHOULD prefer `const` constructors and `const` literals when values are compile-time constants, to reduce allocations and improve canonicalization.
- R077: SHOULD avoid per-call allocations in hot paths (for example, repeatedly allocating `RegExp`, `DateFormat`, or large temporary collections); move reusable objects to `final` top-level or cached singletons.
- R078: If collection size is known, SHOULD preallocate (for example, `List.filled` or `List.generate`) instead of repeated growth in a loop.
- R079: MUST justify any micro-optimization that reduces readability with a benchmark or profiler evidence (see Benchmark rules). [S42]
- R080: MUST treat resource ownership explicitly: the code that creates a resource is responsible for closing/disposing it, unless ownership is transferred and documented.
- R081: MUST close `Sink`s (including `IOSink`) that your code creates; leaving them open is a defect and is lintable. [S35]
- R082: MUST cancel `StreamSubscription`s that your code creates; leaving them active is a defect and is lintable. [S34]
- R083: MUST use `try`/`finally` (or `using`-style helper) around resources that must be released even on error (files, sockets, subscriptions, ports).
- R084: MUST NOT rely on garbage collection for releasing external resources; explicit `close()`/`dispose()` is required.
- R085: MAY use `Finalizer`/`NativeFinalizer` only as a last-resort safety net for external resources; Finalizers are not guaranteed to run and MUST NOT be the primary cleanup path. [S39][S40]
- R086: When investigating leaks or excessive allocations, SHOULD use DevTools memory tooling and MUST attach evidence to the change that claims to fix a leak. [S38]
- R087: MUST treat all external input (JSON, query params, headers, files, platform messages) as untrusted and validate it at the boundary before use. [S47]
- R088: MUST use `dart:convert` (`jsonDecode`/`jsonEncode`) for JSON unless there is a documented reason to use an alternate codec. [S44][S43]
- R089: MUST NOT let `dynamic` JSON maps leak past the boundary: decode, validate, and convert to typed domain models immediately. [S45]
- R090: SHOULD implement typed `fromJson` (or equivalent) constructors/factories for models and keep them next to the model type; parsing logic MUST be unit-tested. [S45][S24]
- R091: SHOULD use pattern matching or structured checks to validate JSON shape (keys, types, optionality) rather than assuming the shape. [S45]
- R092: For large model graphs or long-lived APIs, SHOULD use code generation (for example `json_serializable`) or an equivalent schema-driven approach to avoid handwritten parsing drift. [S43]
- R093: MUST version serialized formats that cross service boundaries; backwards compatibility rules MUST be documented (added fields optional, removed fields supported for at least one release).
- R094: Custom codecs MUST be deterministic and MUST reject invalid inputs with typed errors that preserve context (field path, reason).
- R095: MUST follow pub package layout conventions (top-level `lib/`, `test/`, `bin/`, `example/`, etc.) unless there is a documented reason not to. [S21]
- R096: MUST keep implementation code under `lib/src/` and keep `lib/` focused on the public API surface. [S22]
- R097: For application packages (runnable apps), MUST commit `pubspec.lock` to source control. [S62][S15]
- R098: For library packages intended to be depended on by others, MUST NOT commit `pubspec.lock` (to allow a range of dependency resolutions). [S15][S62]
- R099: MUST use `dart pub add` to add dependencies (and `dart pub add dev:` for dev dependencies) instead of manual edits, unless automation requires otherwise. [S14]
- R100: MUST run `dart pub get` after any `pubspec.yaml` change and MUST keep dependency resolution reproducible. [S15][S19]
- R101: SHOULD use caret constraints (`^`) for most dependencies and SHOULD avoid overly tight pins unless required for correctness or security. [S14][S18]
- R102: MUST review and minimize `dependency_overrides`; overrides MUST NOT ship in published packages and MUST be removed before release. [S18]
- R103: If using workspaces, SHOULD keep `dependency_overrides` centralized (preferably in the root `pubspec.yaml`) and MUST ensure each overridden package is overridden at most once. [S20][S18]
- R104: MUST use `dart pub outdated` regularly and MUST update dependencies intentionally with tests. [S16]
- R105: MUST run `dart pub upgrade` intentionally (not automatically in every CI run) and MUST review lockfile diffs for applications. [S17][S16]
- R106: For packages published to pub.dev, MUST maintain a CHANGELOG and MUST use `dart pub publish` (or automated publishing) with tags/releases. [S59][S58]
- R107: MUST have automated tests and MUST be runnable via `dart test` for pure Dart packages and via `flutter test` for Flutter packages. [S23]
- R108: MUST place Dart unit tests under `test/` and keep tests deterministic (no real network, no real time, no real randomness without seeding). [S23][S24]
- R109: Flutter projects MUST use the appropriate test level: unit tests for pure logic, widget tests for widgets, and integration tests for full app flows. [S64]
- R110: MUST isolate external dependencies in tests (filesystem, network, databases, platform channels) using fakes or mocks; tests MUST NOT require external services to be available. [S64]
- R111: MUST ensure every bug fix includes (or updates) a regression test that fails before the fix and passes after it.
- R112: SHOULD prefer fakes/stubs over deep mocking when behavior is complex; mocks MUST be limited to interface boundaries and MUST not assert internal implementation details.
- R113: MUST avoid flaky time-based assertions; prefer controlled clocks, fake async, or explicit synchronization points.
- R114: Integration tests MUST be scoped to critical flows and MUST run on CI for at least one representative target environment when shipping apps. [S64]
- R115: CI MUST run `dart pub get`, `dart format -o none --set-exit-if-changed`, `dart analyze --fatal-infos`, and `dart test` (or `flutter test`) on every change. [S6][S7][S15][S23]
- R116: CI MUST fail on any formatting change, analysis warning, or test failure; "warnings allowed" policies MUST NOT be used. [S6][S7]
- R117: CI SHOULD cache the pub cache and build artifacts appropriately, but MUST NOT hide dependency resolution or advisory warnings. [S46]
- R118: Projects SHOULD track coverage and set a ratcheting minimum (coverage must not decrease) rather than a fixed global threshold.
- R119: Monorepos using pub workspaces MUST run CI from the workspace root so analysis and dependency resolution are unified. [S20]
- R120: Release pipelines for published packages SHOULD use automated publishing with tokens stored in CI secret storage, never in the project. [S58][S59]
- R121: MUST NOT hardcode secrets (API keys, tokens, private endpoints) in source, tests, or build scripts; secrets MUST come from a secure runtime configuration or secret manager. [S47]
- R122: MUST treat all external inputs as hostile and validate early; validation failures MUST be handled as controlled errors, not crashes. [S47][S45]
- R123: MUST prefer secure transport (TLS/HTTPS) for network traffic; if non-TLS is required (rare), it MUST be explicitly justified and isolated.
- R124: MUST monitor and respond to dependency security advisories surfaced by the pub client during `dart pub get`. [S46]
- R125: If suppressing an advisory using `ignored_advisories`, MUST document why it is not relevant and MUST revisit the suppression at each dependency update. [S46]
- R126: MUST avoid logging sensitive data (tokens, auth headers, full payloads with personal data). [S47]
- R127: MUST keep `dependency_overrides` out of releases and published artifacts; overrides can hide vulnerable transitive versions. [S18]
- R128: Interop code MUST be isolated behind a small, testable boundary (adapter layer); application layers MUST depend on Dart domain interfaces, not platform-specific APIs. [S54][S52][S48]
- R129: FFI usage MUST be limited to Dart Native platforms; code that must run on web MUST NOT import `dart:ffi`. [S52]
- R130: FFI bindings MUST be generated where possible (for example with `package:ffigen` for Apple interop) and MUST NOT be handwritten at scale. [S53]
- R131: FFI code MUST manage native memory explicitly (allocate and deallocate correctly) and MUST document ownership for every pointer returned or stored. [S52]
- R132: JS interop MUST use the modern interop stack (`dart:js_interop` and related guidance) and SHOULD avoid legacy interop APIs except when maintaining existing code. [S48][S49][S51]
- R133: Web platform API access SHOULD migrate to `package:web` where applicable; new code MUST NOT start on deprecated web interop surfaces. [S50][S49]
- R134: Interop surfaces MUST NOT leak JS/FFI/platform-channel types into public package APIs; wrap them into domain types or extension types that you own. [S57][S48]
- R135: Flutter platform channel messages MUST be versioned and typed; where possible, SHOULD use Pigeon-generated, type-safe channels instead of untyped maps. [S54]
- R136: Platform channel boundaries MUST validate inputs and MUST handle platform exceptions without crashing the Dart isolate. [S54][S27]
- R137: MUST NOT commit `.dart_tool/`, `build/`, or generated `doc/api/` outputs to source control. [S62]
- R138: MUST NOT commit `pubspec.lock` for library packages; MUST NOT delete it for application packages. [S62]
- R139: MUST NOT import from another package’s `lib/src/...`. [S22]
- R140: MUST NOT use `dynamic` (or `Map<String, dynamic>`) as an architectural shortcut across layers (including JSON and isolate messaging). [S45]
- R141: MUST NOT start background Futures without handling errors; unhandled async errors MUST be treated as defects. [S30][S28]
- R142: MUST NOT use `async void` except for UI/event handler entrypoints where the framework requires it; prefer `Future<void>` so failures can be observed.
- R143: MUST NOT use `dependency_overrides` as a long-term solution; overrides MUST be temporary and tracked. [S18]
- R144: MUST NOT ignore pub security advisories without documenting why they are irrelevant. [S46]
- R145: MUST NOT rely on finalizers as deterministic cleanup. [S39]
- R146: MUST NOT use platform channels or JS interop with untyped `Map` payloads as the public boundary; define typed messages and versioning. [S54][S48]
- R147: MUST NOT move CPU-bound work onto the main isolate in Flutter apps in a way that can jank the UI; use isolates or platform-native optimizations. [S32][S33]
- **R001** MUST write code that compiles and runs under **sound null safety**; projects MUST NOT opt out of null safety. [S25]
- **R030** MUST keep the project on sound null safety; nullable types (`T?`) MUST be used intentionally, not as a convenience. [S25]
- **R109** Flutter projects MUST use the appropriate test level: unit tests for pure logic, widget tests for widgets, and integration tests for full app flows. [S64]
- **R120** Release pipelines for published packages SHOULD use automated publishing with tokens stored in CI secret storage, never in the project. [S58][S59]
