# Flutter 2025-2026 Engineering Policy Rules (Flutter 3.x)
This document is a company-neutral policy guide for AI coding agents building and maintaining Flutter applications. It is not intended as a tutorial.
Each rule has an ID (**F###**) and uses normative language (**MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, **MAY**). Each rule is written to be enforceable in code review, static analysis, or CI.
---

## 1. Scope and non-goals

**Why this matters:** A cross-platform Flutter app fails when rules try to cover everything. This ruleset focuses on the common, high-leverage constraints that prevent architecture drift and platform regressions.
- **F001** MUST target **iOS, Android, web, macOS, Windows, Linux** from a single Dart/Flutter project.
- **F002** MUST treat Flutter as the UI runtime for all platforms (no “rewrite per platform” strategy).
- **F003** SHOULD use a single “product app” with internal packages/modules, not multiple divergent apps.
- **F004** MAY maintain small platform-specific shims (native host code, JS glue) when required by platform capabilities.
- **F005** MUST define explicit boundaries between UI, state, data, and platform integration.
- **F006** MUST optimize for maintainability and predictable ownership.
- **F007** MUST NOT treat “cross-platform” as “identical UI everywhere”. Platform-adaptive behavior is expected.
- **F008** Any new platform-specific behavior MUST be implemented behind a platform adapter (not in feature UI code).
Refs:
- Flutter platform integration docs: desktop and web ([Desktop support](https://docs.flutter.dev/platform-integration/desktop), [Web renderers](https://docs.flutter.dev/platform-integration/web/renderers)).
---

## 2. Supported targets and assumptions
**Why this matters:** Performance, security, and capabilities differ across targets. The project must encode assumptions explicitly.
- **F009** MUST declare supported OS/browser baselines in the project (README or `docs/support.md`) and keep them updated per release.
- **F010** MUST run CI tests on at least one representative target per platform family: mobile, web, desktop.
- **F011** MUST assume **Impeller is the default renderer** on iOS and on Android API 29+ in modern Flutter, so rendering behavior can differ from Skia ([Impeller](https://docs.flutter.dev/perf/impeller)).
- **F012** SHOULD assume web builds may be produced in **default** or **WebAssembly** build modes ([Web renderers](https://docs.flutter.dev/platform-integration/web/renderers)).
- **F013** SHOULD treat web as a hostile client: anything shipped to web is inspectable.
- **F014** MAY explicitly support multiple web renderers when business requirements demand it (for example, fallback if Wasm not supported).
- **F015** MUST make platform capability checks explicit and centralized.
- **F016** MUST NOT assume a plugin works on all platforms without verifying.
- **F017** Any new dependency on OS features (filesystem, biometrics, clipboard, notifications, window management) MUST include a platform support matrix (iOS/Android/web/macOS/Windows/Linux) in the PR description.

Refs:
- [Impeller](https://docs.flutter.dev/perf/impeller)
- [Desktop support](https://docs.flutter.dev/platform-integration/desktop)
- [Web renderers](https://docs.flutter.dev/platform-integration/web/renderers)
---

## 3. Project layout rules

**Why this matters:** Large Flutter apps fail when `lib/` becomes a dumping ground. A stable structure enables parallel development and predictable ownership.

- **F018** MUST use a **feature-first** structure for product code. Each feature is self-contained and owns its UI + state + data adapters.
- **F019** MUST place shared cross-cutting code in **explicit** shared packages/modules (`core`, `design_system`, `platform`, `analytics`, etc.), not in random `utils/`.
- **F020** MUST keep platform host directories (`android/`, `ios/`, `web/`, `macos/`, `windows/`, `linux/`) free of business logic. Only platform bootstrapping and configuration belongs there.
- **F021** SHOULD use internal Dart packages (monorepo style) when the app exceeds a small team, or when features need hard boundaries.
- **F022** SHOULD keep test directories co-located with the code they test (feature-oriented tests).
- **F023** MAY use a monorepo tool (for example, Melos) to manage multiple internal packages, if the project actually has multiple packages.

### Recommended layout (opinionated)
- `lib/app/` app composition root (router, DI wiring, app shell, theme wiring)
- `lib/design_system/` design tokens, themes, and shared UI components (the only place allowed to define app visual primitives)
- `lib/features/<feature_name>/` feature modules
  - `presentation/` widgets, pages
  - `state/` controllers/viewmodels/providers
  - `data/` repositories, DTOs, mappers, remote/local data sources
  - `domain/` entities/value objects/use-cases (optional but stable)
- `lib/core/` cross-cutting primitives (logging, errors, result types, time, feature flags)
- `lib/platform/` platform adapters (channels, FFI, web JS interop wrappers)
- `docs/design/` design system docs, token naming, component inventory, UX research notes (links or summaries)
- `test/` mirrors `lib/`
- `integration_test/` end-to-end flows
This aligns with Flutter’s architectural guidance that separates UI and data concerns ([Architecting Flutter apps](https://docs.flutter.dev/app-architecture)).
- **F024** MUST make every feature have a single entrypoint file (`feature.dart`) exporting public types.
- **F025** MUST enforce imports: features may depend on `core/`, `platform/`, and other feature public APIs only.
- **F026** MUST NOT allow features to import private files from other features (no `../features/other_feature/src/...`).
- **F027** A file under `lib/features/<A>/...` MUST NOT import a non-public path from `lib/features/<B>/...` (only import `<B>/feature.dart` or other explicitly exported public APIs).

Refs:
- [Architecting Flutter apps](https://docs.flutter.dev/app-architecture)
- [App architecture case study (package structure)](https://docs.flutter.dev/app-architecture/case-study)
---

## 4. Architecture rules

**Why this matters:** Cross-platform correctness requires stable boundaries. When UI talks to HTTP, storage, or platform APIs directly, the app becomes untestable and fragile.

- **F028** MUST implement **three primary layers**:
  1. **Presentation (UI)**: Widgets only render and forward user intent.
  2. **State/Controller**: Owns view state, coordinates async, calls repositories.
  3. **Data/Platform**: Repositories and platform adapters; no Flutter UI imports.
- **F029** MUST keep all platform integration (platform channels, FFI, JS interop) behind **platform adapters** in `lib/platform/`.
- **F030** MUST ensure dependency direction is inward: UI depends on state, state depends on repositories, repositories depend on low-level clients.
- **F031** SHOULD use immutable domain models and explicit mapping between transport DTOs and domain models.
- **F032** SHOULD use “command” or “use-case” style methods for user intent (for example `submit()`, `refresh()`, `toggleDone()`), not random mutations. This mirrors Flutter’s architecture guidance for testable state management ([Architecture concepts](https://docs.flutter.dev/app-architecture/concepts)).
- **F033** MAY add a domain layer when complexity warrants (validation rules, business policies, cross-feature invariants).
- **F034** MUST treat repositories as the **single source of truth** for data retrieval and caching policy.
- **F035** MUST keep platform adapters narrow with a stable interface.
- **F036** MUST NOT import Flutter UI libraries (`package:flutter/...`) in data layer code.
- **F037** MUST NOT pass `BuildContext` into repositories or services.
- **F038** No class in `lib/features/**/data/` may import `package:flutter/` or `dart:ui`.

Refs:
- [Architecting Flutter apps](https://docs.flutter.dev/app-architecture)
- [Architecture concepts](https://docs.flutter.dev/app-architecture/concepts)
- [Platform channels](https://docs.flutter.dev/platform-integration/platform-channels)
---

## 5. State management rules

**Why this matters:** Uncontrolled state management is the fastest route to tech debt. One app should not be a museum of patterns.

### Chosen direction (opinionated)
Flutter’s docs intentionally list multiple valid approaches without naming a single “best” option ([State management options](https://docs.flutter.dev/data-and-backend/state-mgmt/options)). This ruleset chooses:
- **Default:** Riverpod (current major line in 2025-2026) ([Riverpod](https://riverpod.dev/))
- **Allowed alternative (by exception):** BLoC, only when a feature benefits from event-driven modeling and the team commits to consistent usage ([BLoC library](https://bloclibrary.dev/)).
- **F039** MUST use a **single** primary state management framework across the project (default: Riverpod). Any exception requires a documented decision in the project (ADR).
- **F040** MUST model async state as explicit `loading/data/error` and handle all three in UI.
- **F041** MUST keep state “close” to its feature. No global “god provider” that exposes everything.
- **F042** SHOULD treat state objects as immutable.
- **F043** SHOULD keep side effects in controllers/viewmodels, not in UI widgets.
- **F044** MAY use `setState` only for ephemeral, local UI state that does not affect app behavior outside the current widget subtree (for example, hover state, text field local toggle).
- **F045** MUST use providers as both dependency injection and state wiring when using Riverpod.
- **F046** MUST prefer typed, structured error types (not strings).
- **F047** MUST NOT store `BuildContext` in state.
- **F048** MUST NOT mutate collections in-place in state.
- **F049** Any state that influences navigation, network calls, persistence, auth, or cross-screen behavior MUST NOT be implemented with `setState`.

Refs:
- [State management options](https://docs.flutter.dev/data-and-backend/state-mgmt/options)
- [Riverpod](https://riverpod.dev/)
- [BLoC library](https://bloclibrary.dev/)
---

## 6. UI composition rules

**Why this matters:** Flutter UI performance and maintainability depends on small widgets, controlled rebuilds, and predictable composition.

- **F050** MUST avoid doing expensive work in `build()` (no JSON parsing, no sorting large lists, no synchronous file IO).
- **F051** MUST virtualize long lists and grids (`ListView.builder`, slivers). Never build unbounded child lists eagerly.
- **F052** MUST use `const` constructors wherever possible. Flutter explicitly calls this out as a performance best practice ([Performance best practices](https://docs.flutter.dev/perf/best-practices)).
- **F053** SHOULD prefer creating reusable UI pieces as widgets (`StatelessWidget`) instead of helper functions for better rebuild behavior ([Performance best practices](https://docs.flutter.dev/perf/best-practices)).
- **F054** SHOULD keep widgets under roughly one screen’s worth of complexity. Refactor when a widget exceeds ~200 lines or contains unrelated responsibilities.
- **F055** MAY use `RepaintBoundary` when profiling confirms repaint isolation benefits.
- **F056** MUST keep UI pure: render state, dispatch intents.
- **F057** MUST use adaptive layouts (constraints + input modality) rather than platform branching.
- **F058** MUST NOT call platform channels from UI.
- **F059** MUST NOT trigger network calls from `build()`.
- **F060** Any `ListView` with potentially large item count MUST use a builder/sliver constructor (no `ListView(children: ...)` for dynamic content).

Refs:
- [Performance best practices](https://docs.flutter.dev/perf/best-practices)
- [UI performance profiling](https://docs.flutter.dev/perf/ui-performance)
---

## 6A. Theming, styling, and design system rules

**Why this matters:** Visual inconsistency and ad hoc styling are a primary source of tech debt, accessibility regressions, and cross-platform drift. A centralized design system keeps UI predictable across mobile, web, and desktop while supporting theme changes (light/dark/high contrast) without rewriting screens.

- **F061** MUST maintain a single design system module (`lib/design_system/` or `packages/design_system/`) that owns: `ThemeData`, `ColorScheme`, `TextTheme`, `ThemeExtension` tokens, and shared components.
- **F062** MUST in feature UI code (`lib/features/**/presentation/**`), do not introduce hard-coded visual values:
  - No `Color(0x...)` literals
  - No `TextStyle(...)` constructors for app typography
  - No magic spacing numbers (for example `EdgeInsets.all(17)`)
  Use theme and design tokens only.
- **F063** MUST define and ship both `theme` and `darkTheme`, and set `themeMode` explicitly on `MaterialApp.router` (default: `ThemeMode.system`).
- **F064** MUST use `ColorScheme` roles as the source of truth for colors. Do not use deprecated roles (`background`, `onBackground`, etc.) and do not rely on `ThemeData.primaryColor` for new code.
- **F065** MUST use `ThemeExtension` for non-Material tokens (spacing scale, radii, shadows, motion durations, semantic colors). Every `ThemeExtension` MUST implement `copyWith` and `lerp`, and MUST be registered via `ThemeData.extensions`.
- **F066** MUST all reusable UI components (buttons, text fields, cards, dialogs, banners, toasts) MUST live in the design system module. Features MAY compose them, but MUST NOT fork them.
- **F067** MUST any change to tokens or component styles MUST update both light and dark themes (and high-contrast if supported) and MUST include a visual verification note in the PR.
- **F068** SHOULD generate base color schemes from a small set of brand seed colors (Material Theme Builder or `ColorScheme.fromSeed`) and keep the seed(s) in source control.
- **F069** SHOULD use component themes in `ThemeData` (for example `filledButtonTheme`, `inputDecorationTheme`, `snackBarTheme`) instead of styling widgets ad hoc.
- **F070** SHOULD maintain canonical design tokens in a tool-friendly format (recommended: Design Tokens Community Group JSON format) and generate Dart token code from it.
- **F071** SHOULD use a predictable naming scheme for tokens: semantic first (for example `surface`, `onSurface`, `danger`, `success`, `spacingMd`) rather than raw palette names (for example `blue500`).
- **F072** SHOULD for major UX changes, follow an iterative design loop (prototype, test with users, adjust) and capture findings and decisions in `docs/design/` (or equivalent).
- **F073** MAY support Android 12+ dynamic color ("Material You") using `dynamic_color` if the product wants platform-personalized themes. If enabled, you MUST provide a deterministic fallback scheme and verify all `ColorScheme` roles used by the app are populated and meet contrast expectations.
- **F074** MUST access styling through `Theme.of(context)` and `Theme.of(context).extension<AppTokens>()` (or a `BuildContext` helper) in feature code.
- **F075** MUST keep typography responsive to text scaling by using `TextTheme` roles, not hard-coded font sizes.
- **F076** MUST treat design updates as governed changes: include design ticket link and update the component inventory.
- **F077** MUST NOT sprinkle `Theme(...)` overrides inside random subtrees to "fix" styling. Fix the design system instead.
- **F078** MUST NOT use `Colors.*` or `TextStyle(...)` directly in feature UI code.
- **F079** MUST NOT store design tokens as mutable globals or static singletons outside `ThemeData`.
- **F080** Outside `lib/design_system/**` (and tests), the PR MUST NOT introduce `Color(0x` literals, `TextStyle(` constructors, or new spacing magic numbers.

Refs:
- Flutter theming: [Use themes](https://docs.flutter.dev/cookbook/design/themes), [ThemeData](https://api.flutter.dev/flutter/material/ThemeData-class.html), [MaterialApp.themeMode](https://api.flutter.dev/flutter/material/MaterialApp/themeMode.html).
- Theme extensions: [ThemeExtension](https://api.flutter.dev/flutter/material/ThemeExtension-class.html), [ThemeData.extensions](https://api.flutter.dev/flutter/material/ThemeData/extensions.html).
- Material 3 theming changes: [Material 3 default](https://docs.flutter.dev/release/breaking-changes/material-3-default), [New ColorScheme roles](https://docs.flutter.dev/release/breaking-changes/new-color-scheme-roles).
- Typography guidance: [Flutter typography](https://docs.flutter.dev/ui/design/text/typography), [Apple typography](https://developer.apple.com/design/human-interface-guidelines/typography).
- Design tokens standard: [DTCG Design Tokens Format 2025.10](https://www.designtokens.org/TR/2025.10/format/).
- Dynamic color: [Android Dynamic Color](https://developer.android.com/develop/ui/views/theming/dynamic-colors), [dynamic_color package](https://pub.dev/packages/dynamic_color).
- Design research: [Iterative design](https://www.nngroup.com/articles/parallel-and-iterative-design/), [Usability testing 101](https://www.nngroup.com/articles/usability-testing-101/).
---

## 7. Navigation and routing rules

**Why this matters:** Routing must work for mobile back behavior, web deep links, and desktop windowing expectations. Hand-rolled navigation quickly becomes inconsistent.

- **F081** MUST use the Router API (Navigator 2.0 family) through an approved routing package (default: `go_router`) ([go_router](https://pub.dev/packages/go_router)).
- **F082** MUST make deep links first-class: every user-facing screen that can be reached from outside the app must have a stable route.
- **F083** MUST for web, configure URL strategy intentionally (path vs hash) and configure server rewrites when using path URLs ([URL strategy](https://docs.flutter.dev/ui/navigation/url-strategies)).
- **F084** SHOULD keep routing configuration centralized in `lib/app/router.dart`, with features registering routes via a constrained API.
- **F085** SHOULD prefer typed route parameters or explicit parsing functions to avoid runtime errors.
- **F086** MAY use nested navigation stacks per tab/section when UX requires it.
- **F087** MUST call `usePathUrlStrategy()` before `runApp` if you choose path URLs on web ([URL strategy](https://docs.flutter.dev/ui/navigation/url-strategies)).
- **F088** MUST update `<base href>` when hosting at a non-root path ([URL strategy](https://docs.flutter.dev/ui/navigation/url-strategies)).
- **F089** MUST NOT push routes with raw string concatenation for params (encode and validate).
- **F090** MUST NOT disable browser back/forward behavior on web.
- **F091** Any new route added MUST include: (1) a named route entry, (2) parameter validation, and (3) a deep-link test case or documented manual test steps for web.

Refs:
- [URL strategy](https://docs.flutter.dev/ui/navigation/url-strategies)
- [go_router](https://pub.dev/packages/go_router)
---

## 8. Async, isolates, and concurrency rules

**Why this matters:** The UI thread is shared across platforms. Blocking it causes jank everywhere. Web and desktop add extra constraints around threading and IO.

- **F092** MUST avoid blocking the main isolate. No heavy CPU work (large JSON parsing, crypto, image processing) on the UI isolate.
- **F093** MUST add cancellation/timeouts to network operations and long-running tasks.
- **F094** MUST treat all async errors as data: they must be caught and surfaced as state, not swallowed.
- **F095** SHOULD offload CPU-heavy work using isolates where supported and where it measurably improves responsiveness.
- **F096** SHOULD prefer chunking work or moving it to the backend for web when isolate support or transferability limits apply.
- **F097** MAY use background isolates for non-UI workloads when the data passed between isolates is simple and transferable.
- **F098** MUST use `AsyncValue.guard` (or equivalent) to standardize error handling in controllers.
- **F099** MUST use `package:flutter/foundation.dart` `compute()` only for pure functions and transferable data.
- **F100** MUST NOT rely on isolates as a blanket solution for web without validating behavior in the chosen web build mode.
- **F101** Any operation that can exceed ~8ms in worst case MUST NOT run synchronously in `build()`, gesture handlers, or animation callbacks.

Refs:
- [UI performance profiling](https://docs.flutter.dev/perf/ui-performance)
- [Performance best practices](https://docs.flutter.dev/perf/best-practices)
---

## 9. Performance rules (global)

**Why this matters:** Performance regressions are easiest to introduce and hardest to fix late. Enforce rules that prevent jank and runaway memory usage.

- **F102** MUST measure performance in **profile or release** mode, not debug, because debug mode is intentionally slow ([Build modes](https://docs.flutter.dev/testing/build-modes)).
- **F103** MUST use DevTools Performance view to validate frame timing when introducing animations, large lists, or heavy screens ([DevTools Performance view](https://docs.flutter.dev/tools/devtools/performance)).
- **F104** MUST keep app size under active control and review bundle changes for web ([Measuring app size](https://docs.flutter.dev/perf/app-size)).
- **F105** SHOULD prefer `const` widgets and small widget subtrees for rebuild control ([Performance best practices](https://docs.flutter.dev/perf/best-practices)).
- **F106** SHOULD use image resizing (server-side or client-side decode sizing) and avoid decoding full-resolution assets unnecessarily.
- **F107** MAY use deferred loading/components when initial download size is a user-visible problem ([Deferred components](https://docs.flutter.dev/perf/deferred-components)).
- **F108** MUST use `ListView.builder`/slivers, pagination, and caching.
- **F109** MUST keep animations cheap: avoid unnecessary opacity layers and large repaints.
- **F110** MUST NOT ship performance changes without at least one profiling screenshot or metric when the change is large.
- **F111** PRs that add a new animation, large list, or image-heavy screen MUST include a note describing how performance was validated (DevTools trace, profile build, or equivalent).

Refs:
- [Build modes](https://docs.flutter.dev/testing/build-modes)
- [DevTools Performance view](https://docs.flutter.dev/tools/devtools/performance)
- [Performance best practices](https://docs.flutter.dev/perf/best-practices)
- [App size](https://docs.flutter.dev/perf/app-size)
---

## 10. Web-specific rules

**Why this matters:** Flutter web has unique constraints: bundle size, renderer differences, browser security policies, and URL behavior.

- **F112** MUST keep web-only code behind a platform adapter. No web imports in feature UI/state unless it is strictly presentational.
- **F113** MUST configure URL strategy intentionally and configure hosting rewrites for deep links when using path URLs ([URL strategy](https://docs.flutter.dev/ui/navigation/url-strategies)).
- **F114** MUST if customizing web startup behavior, do it via `flutter_bootstrap.js` in `web/` and document the reason ([Web initialization](https://docs.flutter.dev/platform-integration/web/initialization)).
- **F115** SHOULD consider WebAssembly build mode when supported by dependencies, and ensure JS interop uses modern APIs (`dart:js_interop`, `package:web`) as required by Flutter’s WebAssembly guidance ([Web renderers](https://docs.flutter.dev/platform-integration/web/renderers)).
- **F116** SHOULD treat service worker caching as a product decision. Understand update semantics and cache busting ([Web initialization](https://docs.flutter.dev/platform-integration/web/initialization), [Web loading speed blog](https://blog.flutter.dev/best-practices-for-optimizing-flutter-web-loading-speed-7cc0df14ce5c)).
- **F117** MAY use deferred imports/deferred components to reduce initial JS payload ([Deferred components](https://docs.flutter.dev/perf/deferred-components)).
- **F118** MUST validate web release builds via a local web server, not opening files from disk ([Build and release web](https://docs.flutter.dev/deployment/web)).
- **F119** MUST profile web performance with the recommended tools ([Web performance](https://docs.flutter.dev/perf/web-performance)).
- **F120** MUST NOT store long-lived auth tokens in browser localStorage unless you have no alternative and have a documented risk acceptance. Prefer secure cookie-based sessions when feasible (see OWASP session guidance).
- **F121** MUST NOT assume a renderer: test with the chosen renderer and document it ([Web renderers](https://docs.flutter.dev/platform-integration/web/renderers)).
- **F122** Any new web-only capability MUST have (1) a platform adapter in `lib/platform/` and (2) a documented hosting/config requirement if it needs headers, rewrites, or HTTPS.

Refs:
- [Web renderers](https://docs.flutter.dev/platform-integration/web/renderers)
- [Web initialization](https://docs.flutter.dev/platform-integration/web/initialization)
- [URL strategy](https://docs.flutter.dev/ui/navigation/url-strategies)
- [Web performance](https://docs.flutter.dev/perf/web-performance)
- [OWASP Session Management](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
---

## 11. Desktop-specific rules

**Why this matters:** Desktop users expect keyboard shortcuts, focus traversal, window resizing, and OS-native distribution workflows.

- **F123** MUST support keyboard navigation and focus traversal for all interactive UI on desktop (and web) ([User input & accessibility](https://docs.flutter.dev/ui/adaptive-responsive/input)).
- **F124** MUST treat window size as dynamic: layouts MUST be responsive and not assume phone constraints.
- **F125** MUST ensure all plugins used by desktop builds support the target desktop OS or provide a stub/fallback.
- **F126** SHOULD implement app-wide shortcuts using Flutter’s Actions/Shortcuts system ([Actions and shortcuts](https://docs.flutter.dev/ui/interactivity/actions-and-shortcuts)).
- **F127** SHOULD provide desktop-appropriate UI affordances (hover states, right-click context menus, scroll wheel behavior).
- **F128** MAY use platform-native look-and-feel widget sets when product design calls for it ([Building macOS apps](https://docs.flutter.dev/platform-integration/macos/building)).
- **F129** MUST build desktop release artifacts using `flutter build windows|macos|linux` ([Desktop support](https://docs.flutter.dev/platform-integration/desktop)).
- **F130** MUST follow OS distribution requirements (notarization on macOS, packaging on Windows, bundle completeness on Linux) ([Building macOS apps](https://docs.flutter.dev/platform-integration/macos/building), [Building Windows apps](https://docs.flutter.dev/platform-integration/windows/building), [Build Linux apps](https://docs.flutter.dev/platform-integration/linux/building)).
- **F131** MUST NOT ship desktop UI that requires a mouse for basic flows.
- **F132** Any new interactive control MUST be reachable via keyboard (Tab traversal) and activatable via Enter/Space on desktop/web.

Refs:
- [Desktop support](https://docs.flutter.dev/platform-integration/desktop)
- [User input & accessibility](https://docs.flutter.dev/ui/adaptive-responsive/input)
- [Building Windows apps](https://docs.flutter.dev/platform-integration/windows/building)
- [Building macOS apps](https://docs.flutter.dev/platform-integration/macos/building)
- [Build Linux apps](https://docs.flutter.dev/platform-integration/linux/building)
---

## 12. Mobile-specific rules

**Why this matters:** Mobile platforms have strict lifecycle, background execution, and store policies. Flutter abstractions do not remove these constraints.

- **F133** MUST handle app lifecycle events for any feature that depends on foreground/background state (audio, timers, auth refresh, camera).
- **F134** MUST ensure navigation respects platform conventions (system back on Android, swipe back where applicable).
- **F135** MUST use platform-appropriate secure storage for sensitive user secrets (see Security rules).
- **F136** SHOULD validate store requirements early (permissions, privacy disclosures, entitlements) ([Release iOS](https://docs.flutter.dev/deployment/ios), [Release Android](https://docs.flutter.dev/deployment/android)).
- **F137** SHOULD keep startup time low by deferring non-critical initialization.
- **F138** MAY use Android dynamic feature modules for large apps (deferred components) when install size is a major issue ([Deferred components](https://docs.flutter.dev/perf/deferred-components)).
- **F139** MUST use release builds for validation ([Build modes](https://docs.flutter.dev/testing/build-modes)).
- **F140** MUST use TestFlight/internal testing tracks before full release.
- **F141** MUST NOT request broad permissions without a user-visible feature and a documented reason.
- **F142** Any new permission (camera, location, notifications, storage) MUST include (1) a user-facing explanation string, and (2) a documented justification in the PR.

Refs:
- [Build and release iOS](https://docs.flutter.dev/deployment/ios)
- [Build and release Android](https://docs.flutter.dev/deployment/android)
- [Deferred components](https://docs.flutter.dev/perf/deferred-components)
---

## 13. Networking and caching rules

**Why this matters:** Networking code is a cross-cutting risk area: performance, reliability, observability, and security all depend on it.

- **F143** MUST centralize HTTP client configuration (timeouts, headers, auth, retry policy) in one module (`core/networking`).
- **F144** MUST avoid direct HTTP calls in widgets or UI state objects.
- **F145** MUST treat caching as a project concern with explicit TTL/invalidations.
- **F146** SHOULD use idempotent retry with backoff for safe operations only.
- **F147** SHOULD design API clients so they can be mocked in unit tests.
- **F148** MAY use a local cache (database or file cache) for offline and large data, if product needs it.
- **F149** MUST use DevTools Network view when diagnosing networking performance ([DevTools Network view](https://docs.flutter.dev/tools/devtools/network)).
- **F150** MUST NOT log full request/response bodies when they can include PII or secrets.
- **F151** New endpoints MUST be integrated through a project method that can be unit tested without Flutter widgets.

Refs:
- [DevTools Network view](https://docs.flutter.dev/tools/devtools/network)
---

## 14. Storage and persistence rules

**Why this matters:** Storage APIs differ across platforms. Web storage is not secure. Desktop has real filesystem access. Mobile has sandboxed storage and OS keychains.

- **F152** MUST use key-value storage only for small, non-sensitive preferences.
- **F153** MUST use secure storage for sensitive user secrets (tokens), with explicit platform constraints.
- **F154** MUST abstract persistence behind repositories so UI does not depend on storage backend.
- **F155** SHOULD use a real database for structured data and queries (instead of stuffing JSON blobs into preferences).
- **F156** SHOULD design migration strategy for any persisted schema.
- **F157** MAY use platform-specific storage for desktop integrations (file pickers, recent documents) behind a platform adapter.
- **F158** MUST use `shared_preferences` only for preferences and small settings ([shared_preferences](https://pub.dev/packages/shared_preferences)).
- **F159** MUST use `flutter_secure_storage` for secrets where appropriate, understanding web limitations (HTTPS requirement, WebCrypto behavior) ([flutter_secure_storage](https://github.com/mogol/flutter_secure_storage)).
- **F160** MUST NOT assume secure storage on web provides the same protection as OS keychains.
- **F161** No auth token or refresh token may be stored in `shared_preferences`.

Refs:
- [shared_preferences](https://pub.dev/packages/shared_preferences)
- [flutter_secure_storage](https://github.com/mogol/flutter_secure_storage)
---

## 15. Security rules

**Why this matters:** The same Flutter code ships to very different threat models. Web is fully inspectable; mobile/desktop can be reverse engineered. Security must be layered and pragmatic.

- **F162** MUST never embed server secrets or privileged API keys in the client. Assume web builds are public.
- **F163** MUST ensure all network communication uses TLS and fails closed on invalid certificates (default platform behavior; do not bypass).
- **F164** MUST centralize auth/session handling so token refresh, logout, and revocation are consistent.
- **F165** SHOULD prefer cookie-based web sessions with secure attributes when feasible, following OWASP guidance.
- **F166** SHOULD use code obfuscation and split debug info for release builds when the product requires it ([Obfuscate Dart code](https://docs.flutter.dev/deployment/obfuscate)).
- **F167** SHOULD avoid custom cryptography. Use platform-provided primitives (WebCrypto, Keychain/Keystore) or vetted libraries.
- **F168** MAY add platform-specific hardening (root/jailbreak signals, jailbreak detection) if required by the product threat model.
- **F169** MUST capture and report uncaught errors using both framework and platform-level hooks ([Handle errors](https://docs.flutter.dev/testing/errors), [PlatformDispatcher.onError](https://api.flutter.dev/flutter/dart-ui/PlatformDispatcher/onError.html)).
- **F170** MUST maintain a dependency vulnerability review process.
- **F171** MUST NOT store long-lived tokens in web local storage without a risk acceptance (OWASP).
- **F172** MUST NOT log secrets.
- **F173** Any new configuration value added to the project MUST be classified as public or secret. Secrets MUST NOT be committed and MUST be injected via CI/runtime.

Refs:
- [Obfuscate Dart code](https://docs.flutter.dev/deployment/obfuscate)
- [OWASP MASVS](https://mas.owasp.org/)
- [OWASP Session Management](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
---

## 16. Testing rules

**Why this matters:** A cross-platform app requires a layered test strategy. Tests must be fast, deterministic, and target the right layer.

- **F174** MUST unit test repositories, pure functions, and business rules without Flutter.
- **F175** MUST widget test UI components that contain logic (conditional rendering, validation, error states).
- **F176** MUST maintain at least one integration test per critical user journey.
- **F177** SHOULD avoid golden tests for highly dynamic UI and for platform-dependent rendering differences. Use them for stable design-system components only.
- **F178** SHOULD run tests in CI for every PR.
- **F179** MAY use platform-specific integration tests when a feature is platform-specific (desktop file dialogs, web auth flows).
- **F180** MUST prefer integration testing via the `integration_test` package and profile when needed ([Integration tests](https://docs.flutter.dev/testing/integration-tests)).
- **F181** MUST keep tests hermetic: mock network and time.
- **F182** MUST NOT rely on manual testing as the only validation for cross-platform behavior.
- **F183** Any bug fix MUST include a regression test at the lowest practical layer (unit first, widget next, integration last).

Refs:
- [Integration tests](https://docs.flutter.dev/testing/integration-tests)
---

## 17. Observability rules (logging/metrics/tracing)

**Why this matters:** Multi-platform issues are hard to reproduce. Observability must be built in and consistent.

- **F184** MUST use structured logging (category/name + fields). No raw `print()` in production code.
- **F185** MUST route uncaught framework and async errors to a central reporting pipeline ([Handle errors](https://docs.flutter.dev/testing/errors), [PlatformDispatcher.onError](https://api.flutter.dev/flutter/dart-ui/PlatformDispatcher/onError.html)).
- **F186** MUST ensure logs are useful in DevTools (and in your production log sink) ([DevTools Logging view](https://docs.flutter.dev/tools/devtools/logging)).
- **F187** SHOULD add basic metrics around startup time, screen load time, and major network operations.
- **F188** SHOULD correlate network requests with user actions (trace IDs) where feasible.
- **F189** MAY add distributed tracing if the backend supports it and the product requires it.
- **F190** MUST use DevTools Performance and Logging views during development ([DevTools Performance view](https://docs.flutter.dev/tools/devtools/performance), [DevTools Logging view](https://docs.flutter.dev/tools/devtools/logging)).
- **F191** MUST NOT log PII, auth tokens, or secrets.
- **F192** New error handling code MUST include (1) a user-facing error state and (2) a log/report event that excludes sensitive data.

Refs:
- [DevTools Logging view](https://docs.flutter.dev/tools/devtools/logging)
- [Handle errors](https://docs.flutter.dev/testing/errors)
- [PlatformDispatcher.onError](https://api.flutter.dev/flutter/dart-ui/PlatformDispatcher/onError.html)
---

## 18. Accessibility rules

**Why this matters:** Accessibility is not optional. Desktop and web also require keyboard and focus correctness.

- **F193** MUST all interactive controls MUST be reachable by keyboard and expose accessible labels where needed ([User input & accessibility](https://docs.flutter.dev/ui/adaptive-responsive/input)).
- **F194** MUST use semantic widgets correctly (buttons are buttons, not gesture detectors on containers).
- **F195** MUST ensure focus order is logical and stable.
- **F196** SHOULD test with screen readers on mobile (VoiceOver/TalkBack) and keyboard-only navigation on desktop/web.
- **F197** SHOULD follow web accessibility guidance for Flutter web ([Web accessibility](https://docs.flutter.dev/platform-integration/web/web-accessibility)).
- **F198** MAY add custom Semantics nodes for complex controls.
- **F199** MUST use `FocusableActionDetector` for custom controls that need focus, mouse, and shortcuts ([User input & accessibility](https://docs.flutter.dev/ui/adaptive-responsive/input)).
- **F200** MUST NOT create custom clickable widgets that are not in the focus traversal order.
- **F201** Any new custom interactive widget MUST include Semantics and be focusable with Tab.

Refs:
- [User input & accessibility](https://docs.flutter.dev/ui/adaptive-responsive/input)
- [Web accessibility](https://docs.flutter.dev/platform-integration/web/web-accessibility)
---

## 19. Internationalization rules

**Why this matters:** i18n done late is expensive. Flutter provides a standardized workflow that should be used consistently.

- **F202** MUST no user-facing strings may be hard-coded in widgets outside of localization files.
- **F203** MUST use Flutter’s localization tooling (`l10n.yaml`, ARB files, generated localizations) ([Internationalization](https://docs.flutter.dev/ui/internationalization)).
- **F204** SHOULD include pluralization and parameter typing in ARB placeholders.
- **F205** SHOULD ensure locale changes do not require app restart.
- **F206** MAY add per-locale formatting utilities for dates/numbers in a shared module.
- **F207** MUST enable `flutter: generate: true` and commit the ARB source files (not generated outputs) ([Internationalization](https://docs.flutter.dev/ui/internationalization)).
- **F208** MUST NOT concatenate strings for sentences (breaks translation).
- **F209** Any new user-visible text in UI MUST be sourced from generated localizations (no raw string literals).

Refs:
- [Internationalization](https://docs.flutter.dev/ui/internationalization)
---

## 20. Release and CI rules

**Why this matters:** Release practices must account for different build pipelines and store requirements. CI is the enforcement mechanism for this ruleset.

- **F210** MUST CI MUST run: formatting, `flutter analyze`, unit/widget tests, and at least a smoke integration test.
- **F211** MUST all shipped artifacts MUST be built in release mode ([Build modes](https://docs.flutter.dev/testing/build-modes)).
- **F212** MUST release processes MUST follow platform guidance for each target ([iOS release](https://docs.flutter.dev/deployment/ios), [Android release](https://docs.flutter.dev/deployment/android), [Web release](https://docs.flutter.dev/deployment/web), [Windows release](https://docs.flutter.dev/deployment/windows), [macOS release](https://docs.flutter.dev/deployment/macos), [Linux release](https://docs.flutter.dev/deployment/linux)).
- **F213** SHOULD lock the Flutter SDK version used by CI and developers (use a version manager if needed) to avoid “works on my machine” drift.
- **F214** SHOULD use obfuscation + split debug info when required, and store symbol files securely ([Obfuscation](https://docs.flutter.dev/deployment/obfuscate)).
- **F215** MAY automate store uploads and signing in CI (Codemagic/GitHub Actions/etc.) if security requirements are met.
- **F216** MUST validate web builds using a local server before deployment ([Web release](https://docs.flutter.dev/deployment/web)).
- **F217** MUST validate desktop packaging on a clean machine or VM.
- **F218** MUST NOT ship debug/profile builds to end users.
- **F219** Any change to build configuration (Gradle/Xcode/web bootstrap) MUST include CI updates and a documented manual validation step.

Refs:
- [Build modes](https://docs.flutter.dev/testing/build-modes)
- [Deployment: web](https://docs.flutter.dev/deployment/web)
---

## 21. Anti-patterns (explicit do-not-do list)

**Why this matters:** These are common failure modes that create long-term maintenance cost.
- **F220** MUST NOT perform network calls, database IO, file IO, or platform channel calls inside `build()`.
- **F221** MUST NOT store auth tokens in `shared_preferences`.
- **F222** MUST NOT scatter `kIsWeb` / `Platform.isX` checks across feature code.
- **F223** MUST NOT hard-code visual styling (colors, typography, spacing, radii) outside `lib/design_system/**`.
- **F224** MUST NOT import `dart:io` in code that is compiled for web.
- **F225** MUST NOT introduce a second state management framework without an ADR and migration plan.
- **F226** MUST NOT catch errors and ignore them (“empty catch”).
- **F227** MUST NOT use `print()` as production logging.
- **F228** MUST NOT add dependencies that do not support all target platforms without a documented fallback.
- **F229** MUST NOT build large dynamic lists with `ListView(children: ...)` instead of virtualization.
- **F230** SHOULD NOT use golden tests as the primary UI validation for cross-platform screens.
- **F231** SHOULD NOT hard-code strings in UI widgets.
- **F232** SHOULD NOT use mutable shared singletons as “global state”.
- **F233** MUST NOT implement custom cryptography.
- **F234** MUST NOT bypass TLS validation.
- **F235** If a PR introduces any of the MUST NOT items, it fails review unless the PR is explicitly a migration away from that anti-pattern.

Refs:
- [Performance best practices](https://docs.flutter.dev/perf/best-practices)
- [URL strategy](https://docs.flutter.dev/ui/navigation/url-strategies)
- [Platform channels](https://docs.flutter.dev/platform-integration/platform-channels)


## Additional Merged Rules

- **MUST**: required for all production code.
- **SHOULD**: required unless you have a documented reason to diverge.
- **MAY**: optional, use when it clearly improves the outcome.
- Target **iOS, Android, web, macOS, Windows, Linux** from a single Dart/Flutter codebase.
- Treat Flutter as the UI runtime for all platforms (no “rewrite per platform” strategy).
- Use a single “product app” with internal packages/modules, not multiple divergent apps.
- Maintain small platform-specific shims (native host code, JS glue) when required by platform capabilities.
- Define explicit boundaries between UI, state, data, and platform integration.
- Optimize for maintainability and predictable ownership.
- Don’t treat “cross-platform” as “identical UI everywhere”. Platform-adaptive behavior is expected.
- **PR CHECK:** Any new platform-specific behavior MUST be implemented behind a platform adapter (not in feature UI code).
- Declare supported OS/browser baselines in the repo (README or `docs/support.md`) and keep them updated per release.
- Run CI tests on at least one representative target per platform family: mobile, web, desktop.
- Assume **Impeller is the default renderer** on iOS and on Android API 29+ in modern Flutter, so rendering behavior can differ from Skia ([Impeller](https://docs.flutter.dev/perf/impeller)).
- Assume web builds may be produced in **default** or **WebAssembly** build modes ([Web renderers](https://docs.flutter.dev/platform-integration/web/renderers)).
- Treat web as a hostile client: anything shipped to web is inspectable.
- Explicitly support multiple web renderers when business requirements demand it (for example, fallback if Wasm not supported).
- Make platform capability checks explicit and centralized.
- Don’t assume a plugin works on all platforms without verifying.
- **PR CHECK:** Any new dependency on OS features (filesystem, biometrics, clipboard, notifications, window management) MUST include a platform support matrix (iOS/Android/web/macOS/Windows/Linux) in the PR description.
- Use a **feature-first** structure for product code. Each feature is self-contained and owns its UI + state + data adapters.
- Place shared cross-cutting code in **explicit** shared packages/modules (`core`, `design_system`, `platform`, `analytics`, etc.), not in random `utils/`.
- Keep platform host directories (`android/`, `ios/`, `web/`, `macos/`, `windows/`, `linux/`) free of business logic. Only platform bootstrapping and configuration belongs there.
- Use internal Dart packages (monorepo style) when the app exceeds a small team, or when features need hard boundaries.
- Keep test directories co-located with the code they test (feature-oriented tests).
- Use a monorepo tool (for example, Melos) to manage multiple internal packages, if the repo actually has multiple packages.
- Make every feature have a single entrypoint file (`feature.dart`) exporting public types.
- Enforce imports: features may depend on `core/`, `platform/`, and other feature public APIs only.
- Don’t allow features to import private files from other features (no `../features/other_feature/src/...`).
- **PR CHECK:** A file under `lib/features/<A>/...` MUST NOT import a non-public path from `lib/features/<B>/...` (only import `<B>/feature.dart` or other explicitly exported public APIs).
- Implement **three primary layers**:
- Keep all platform integration (platform channels, FFI, JS interop) behind **platform adapters** in `lib/platform/`.
- Ensure dependency direction is inward: UI depends on state, state depends on repositories, repositories depend on low-level clients.
- Use immutable domain models and explicit mapping between transport DTOs and domain models.
- Use “command” or “use-case” style methods for user intent (for example `submit()`, `refresh()`, `toggleDone()`), not random mutations. This mirrors Flutter’s architecture guidance for testable state management ([Architecture concepts](https://docs.flutter.dev/app-architecture/concepts)).
- Add a domain layer when complexity warrants (validation rules, business policies, cross-feature invariants).
- Treat repositories as the **single source of truth** for data retrieval and caching policy.
- Keep platform adapters narrow with a stable interface.
- Don’t import Flutter UI libraries (`package:flutter/...`) in data layer code.
- Don’t pass `BuildContext` into repositories or services.
- **PR CHECK:** No class in `lib/features/**/data/` may import `package:flutter/` or `dart:ui`.
- Use a **single** primary state management framework across the codebase (default: Riverpod). Any exception requires a documented decision in the repo (ADR).
- Model async state as explicit `loading/data/error` and handle all three in UI.
- Keep state “close” to its feature. No global “god provider” that exposes everything.
- Treat state objects as immutable.
- Keep side effects in controllers/viewmodels, not in UI widgets.
- Use `setState` only for ephemeral, local UI state that does not affect app behavior outside the current widget subtree (for example, hover state, text field local toggle).
- Use providers as both dependency injection and state wiring when using Riverpod.
- Prefer typed, structured error types (not strings).
- Don’t store `BuildContext` in state.
- Don’t mutate collections in-place in state.
- **PR CHECK:** Any state that influences navigation, network calls, persistence, auth, or cross-screen behavior MUST NOT be implemented with `setState`.
- Avoid doing expensive work in `build()` (no JSON parsing, no sorting large lists, no synchronous file IO).
- Virtualize long lists and grids (`ListView.builder`, slivers). Never build unbounded child lists eagerly.
- Use `const` constructors wherever possible. Flutter explicitly calls this out as a performance best practice ([Performance best practices](https://docs.flutter.dev/perf/best-practices)).
- Prefer creating reusable UI pieces as widgets (`StatelessWidget`) instead of helper functions for better rebuild behavior ([Performance best practices](https://docs.flutter.dev/perf/best-practices)).
- Keep widgets under roughly one screen’s worth of complexity. Refactor when a widget exceeds ~200 lines or contains unrelated responsibilities.
- Use `RepaintBoundary` when profiling confirms repaint isolation benefits.
- Keep UI pure: render state, dispatch intents.
- Use adaptive layouts (constraints + input modality) rather than platform branching.
- Don’t call platform channels from UI.
- Don’t trigger network calls from `build()`.
- **PR CHECK:** Any `ListView` with potentially large item count MUST use a builder/sliver constructor (no `ListView(children: ...)` for dynamic content).
- Maintain a single design system module (`lib/design_system/` or `packages/design_system/`) that owns: `ThemeData`, `ColorScheme`, `TextTheme`, `ThemeExtension` tokens, and shared components.
- In feature UI code (`lib/features/**/presentation/**`), do not introduce hard-coded visual values:
- Define and ship both `theme` and `darkTheme`, and set `themeMode` explicitly on `MaterialApp.router` (default: `ThemeMode.system`).
- Use `ColorScheme` roles as the source of truth for colors. Do not use deprecated roles (`background`, `onBackground`, etc.) and do not rely on `ThemeData.primaryColor` for new code.
- Use `ThemeExtension` for non-Material tokens (spacing scale, radii, shadows, motion durations, semantic colors). Every `ThemeExtension` MUST implement `copyWith` and `lerp`, and MUST be registered via `ThemeData.extensions`.
- All reusable UI components (buttons, text fields, cards, dialogs, banners, toasts) MUST live in the design system module. Features MAY compose them, but MUST NOT fork them.
- Any change to tokens or component styles MUST update both light and dark themes (and high-contrast if supported) and MUST include a visual verification note in the PR.
- Generate base color schemes from a small set of brand seed colors (Material Theme Builder or `ColorScheme.fromSeed`) and keep the seed(s) in source control.
- Use component themes in `ThemeData` (for example `filledButtonTheme`, `inputDecorationTheme`, `snackBarTheme`) instead of styling widgets ad hoc.
- Maintain canonical design tokens in a tool-friendly format (recommended: Design Tokens Community Group JSON format) and generate Dart token code from it.
- Use a predictable naming scheme for tokens: semantic first (for example `surface`, `onSurface`, `danger`, `success`, `spacingMd`) rather than raw palette names (for example `blue500`).
- For major UX changes, follow an iterative design loop (prototype, test with users, adjust) and capture findings and decisions in `docs/design/` (or equivalent).
- Support Android 12+ dynamic color ("Material You") using `dynamic_color` if the product wants platform-personalized themes. If enabled, you MUST provide a deterministic fallback scheme and verify all `ColorScheme` roles used by the app are populated and meet contrast expectations.
- Access styling through `Theme.of(context)` and `Theme.of(context).extension<AppTokens>()` (or a `BuildContext` helper) in feature code.
- Keep typography responsive to text scaling by using `TextTheme` roles, not hard-coded font sizes.
- Treat design updates as governed changes: include design ticket link and update the component inventory.
- Don’t sprinkle `Theme(...)` overrides inside random subtrees to "fix" styling. Fix the design system instead.
- Don’t use `Colors.*` or `TextStyle(...)` directly in feature UI code.
- Don’t store design tokens as mutable globals or static singletons outside `ThemeData`.
- **PR CHECK:** Outside `lib/design_system/**` (and tests), the PR MUST NOT introduce `Color(0x` literals, `TextStyle(` constructors, or new spacing magic numbers.
- Use the Router API (Navigator 2.0 family) through an approved routing package (default: `go_router`) ([go_router](https://pub.dev/packages/go_router)).
- Make deep links first-class: every user-facing screen that can be reached from outside the app must have a stable route.
- For web, configure URL strategy intentionally (path vs hash) and configure server rewrites when using path URLs ([URL strategy](https://docs.flutter.dev/ui/navigation/url-strategies)).
- Keep routing configuration centralized in `lib/app/router.dart`, with features registering routes via a constrained API.
- Prefer typed route parameters or explicit parsing functions to avoid runtime errors.
- Use nested navigation stacks per tab/section when UX requires it.
- Call `usePathUrlStrategy()` before `runApp` if you choose path URLs on web ([URL strategy](https://docs.flutter.dev/ui/navigation/url-strategies)).
- Update `<base href>` when hosting at a non-root path ([URL strategy](https://docs.flutter.dev/ui/navigation/url-strategies)).
- Don’t push routes with raw string concatenation for params (encode and validate).
- Don’t disable browser back/forward behavior on web.
- **PR CHECK:** Any new route added MUST include: (1) a named route entry, (2) parameter validation, and (3) a deep-link test case or documented manual test steps for web.
- Avoid blocking the main isolate. No heavy CPU work (large JSON parsing, crypto, image processing) on the UI isolate.
- Add cancellation/timeouts to network operations and long-running tasks.
- Treat all async errors as data: they must be caught and surfaced as state, not swallowed.
- Offload CPU-heavy work using isolates where supported and where it measurably improves responsiveness.
- Prefer chunking work or moving it to the backend for web when isolate support or transferability limits apply.
- Use background isolates for non-UI workloads when the data passed between isolates is simple and transferable.
- Use `AsyncValue.guard` (or equivalent) to standardize error handling in controllers.
- Use `package:flutter/foundation.dart` `compute()` only for pure functions and transferable data.
- Don’t rely on isolates as a blanket solution for web without validating behavior in the chosen web build mode.
- **PR CHECK:** Any operation that can exceed ~8ms in worst case MUST NOT run synchronously in `build()`, gesture handlers, or animation callbacks.
- Measure performance in **profile or release** mode, not debug, because debug mode is intentionally slow ([Build modes](https://docs.flutter.dev/testing/build-modes)).
- Use DevTools Performance view to validate frame timing when introducing animations, large lists, or heavy screens ([DevTools Performance view](https://docs.flutter.dev/tools/devtools/performance)).
- Keep app size under active control and review bundle changes for web ([Measuring app size](https://docs.flutter.dev/perf/app-size)).
- Prefer `const` widgets and small widget subtrees for rebuild control ([Performance best practices](https://docs.flutter.dev/perf/best-practices)).
- Use image resizing (server-side or client-side decode sizing) and avoid decoding full-resolution assets unnecessarily.
- Use deferred loading/components when initial download size is a user-visible problem ([Deferred components](https://docs.flutter.dev/perf/deferred-components)).
- Use `ListView.builder`/slivers, pagination, and caching.
- Keep animations cheap: avoid unnecessary opacity layers and large repaints.
- Don’t ship performance changes without at least one profiling screenshot or metric when the change is large.
- **PR CHECK:** PRs that add a new animation, large list, or image-heavy screen MUST include a note describing how performance was validated (DevTools trace, profile build, or equivalent).
- Keep web-only code behind a platform adapter. No web imports in feature UI/state unless it is strictly presentational.
- Configure URL strategy intentionally and configure hosting rewrites for deep links when using path URLs ([URL strategy](https://docs.flutter.dev/ui/navigation/url-strategies)).
- If customizing web startup behavior, do it via `flutter_bootstrap.js` in `web/` and document the reason ([Web initialization](https://docs.flutter.dev/platform-integration/web/initialization)).
- Consider WebAssembly build mode when supported by dependencies, and ensure JS interop uses modern APIs (`dart:js_interop`, `package:web`) as required by Flutter’s WebAssembly guidance ([Web renderers](https://docs.flutter.dev/platform-integration/web/renderers)).
- Treat service worker caching as a product decision. Understand update semantics and cache busting ([Web initialization](https://docs.flutter.dev/platform-integration/web/initialization), [Web loading speed blog](https://blog.flutter.dev/best-practices-for-optimizing-flutter-web-loading-speed-7cc0df14ce5c)).
- Use deferred imports/deferred components to reduce initial JS payload ([Deferred components](https://docs.flutter.dev/perf/deferred-components)).
- Validate web release builds via a local web server, not opening files from disk ([Build and release web](https://docs.flutter.dev/deployment/web)).
- Profile web performance with the recommended tools ([Web performance](https://docs.flutter.dev/perf/web-performance)).
- Don’t store long-lived auth tokens in browser localStorage unless you have no alternative and have a documented risk acceptance. Prefer secure cookie-based sessions when feasible (see OWASP session guidance).
- Don’t assume a renderer: test with the chosen renderer and document it ([Web renderers](https://docs.flutter.dev/platform-integration/web/renderers)).
- **PR CHECK:** Any new web-only capability MUST have (1) a platform adapter in `lib/platform/` and (2) a documented hosting/config requirement if it needs headers, rewrites, or HTTPS.
- Support keyboard navigation and focus traversal for all interactive UI on desktop (and web) ([User input & accessibility](https://docs.flutter.dev/ui/adaptive-responsive/input)).
- Treat window size as dynamic: layouts MUST be responsive and not assume phone constraints.
- Ensure all plugins used by desktop builds support the target desktop OS or provide a stub/fallback.
- Implement app-wide shortcuts using Flutter’s Actions/Shortcuts system ([Actions and shortcuts](https://docs.flutter.dev/ui/interactivity/actions-and-shortcuts)).
- Provide desktop-appropriate UI affordances (hover states, right-click context menus, scroll wheel behavior).
- Use platform-native look-and-feel widget sets when product design calls for it ([Building macOS apps](https://docs.flutter.dev/platform-integration/macos/building)).
- Build desktop release artifacts using `flutter build windows|macos|linux` ([Desktop support](https://docs.flutter.dev/platform-integration/desktop)).
- Follow OS distribution requirements (notarization on macOS, packaging on Windows, bundle completeness on Linux) ([Building macOS apps](https://docs.flutter.dev/platform-integration/macos/building), [Building Windows apps](https://docs.flutter.dev/platform-integration/windows/building), [Build Linux apps](https://docs.flutter.dev/platform-integration/linux/building)).
- Don’t ship desktop UI that requires a mouse for basic flows.
- **PR CHECK:** Any new interactive control MUST be reachable via keyboard (Tab traversal) and activatable via Enter/Space on desktop/web.
- Handle app lifecycle events for any feature that depends on foreground/background state (audio, timers, auth refresh, camera).
- Ensure navigation respects platform conventions (system back on Android, swipe back where applicable).
- Use platform-appropriate secure storage for sensitive user secrets (see Security rules).
- Validate store requirements early (permissions, privacy disclosures, entitlements) ([Release iOS](https://docs.flutter.dev/deployment/ios), [Release Android](https://docs.flutter.dev/deployment/android)).
- Keep startup time low by deferring non-critical initialization.
- Use Android dynamic feature modules for large apps (deferred components) when install size is a major issue ([Deferred components](https://docs.flutter.dev/perf/deferred-components)).
- Use release builds for validation ([Build modes](https://docs.flutter.dev/testing/build-modes)).
- Use TestFlight/internal testing tracks before full release.
- Don’t request broad permissions without a user-visible feature and a documented reason.
- **PR CHECK:** Any new permission (camera, location, notifications, storage) MUST include (1) a user-facing explanation string, and (2) a documented justification in the PR.
- Centralize HTTP client configuration (timeouts, headers, auth, retry policy) in one module (`core/networking`).
- Avoid direct HTTP calls in widgets or UI state objects.
- Treat caching as a repository concern with explicit TTL/invalidations.
- Use idempotent retry with backoff for safe operations only.
- Design API clients so they can be mocked in unit tests.
- Use a local cache (database or file cache) for offline and large data, if product needs it.
- Use DevTools Network view when diagnosing networking performance ([DevTools Network view](https://docs.flutter.dev/tools/devtools/network)).
- Don’t log full request/response bodies when they can include PII or secrets.
- **PR CHECK:** New endpoints MUST be integrated through a repository method that can be unit tested without Flutter widgets.
- Use key-value storage only for small, non-sensitive preferences.
- Use secure storage for sensitive user secrets (tokens), with explicit platform constraints.
- Abstract persistence behind repositories so UI does not depend on storage backend.
- Use a real database for structured data and queries (instead of stuffing JSON blobs into preferences).
- Design migration strategy for any persisted schema.
- Use platform-specific storage for desktop integrations (file pickers, recent documents) behind a platform adapter.
- Use `shared_preferences` only for preferences and small settings ([shared_preferences](https://pub.dev/packages/shared_preferences)).
- Use `flutter_secure_storage` for secrets where appropriate, understanding web limitations (HTTPS requirement, WebCrypto behavior) ([flutter_secure_storage](https://github.com/mogol/flutter_secure_storage)).
- Don’t assume secure storage on web provides the same protection as OS keychains.
- **PR CHECK:** No auth token or refresh token may be stored in `shared_preferences`.
- Never embed server secrets or privileged API keys in the client. Assume web builds are public.
- Ensure all network communication uses TLS and fails closed on invalid certificates (default platform behavior; do not bypass).
- Centralize auth/session handling so token refresh, logout, and revocation are consistent.
- Prefer cookie-based web sessions with secure attributes when feasible, following OWASP guidance.
- Use code obfuscation and split debug info for release builds when the product requires it ([Obfuscate Dart code](https://docs.flutter.dev/deployment/obfuscate)).
- Avoid custom cryptography. Use platform-provided primitives (WebCrypto, Keychain/Keystore) or vetted libraries.
- Add platform-specific hardening (root/jailbreak signals, jailbreak detection) if required by the product threat model.
- Capture and report uncaught errors using both framework and platform-level hooks ([Handle errors](https://docs.flutter.dev/testing/errors), [PlatformDispatcher.onError](https://api.flutter.dev/flutter/dart-ui/PlatformDispatcher/onError.html)).
- Maintain a dependency vulnerability review process.
- Don’t store long-lived tokens in web local storage without a risk acceptance (OWASP).
- Don’t log secrets.
- **PR CHECK:** Any new configuration value added to the repo MUST be classified as public or secret. Secrets MUST NOT be committed and MUST be injected via CI/runtime.
- Unit test repositories, pure functions, and business rules without Flutter.
- Widget test UI components that contain logic (conditional rendering, validation, error states).
- Maintain at least one integration test per critical user journey.
- Avoid golden tests for highly dynamic UI and for platform-dependent rendering differences. Use them for stable design-system components only.
- Run tests in CI for every PR.
- Use platform-specific integration tests when a feature is platform-specific (desktop file dialogs, web auth flows).
- Prefer integration testing via the `integration_test` package and profile when needed ([Integration tests](https://docs.flutter.dev/testing/integration-tests)).
- Keep tests hermetic: mock network and time.
- Don’t rely on manual testing as the only validation for cross-platform behavior.
- **PR CHECK:** Any bug fix MUST include a regression test at the lowest practical layer (unit first, widget next, integration last).
- Use structured logging (category/name + fields). No raw `print()` in production code.
- Route uncaught framework and async errors to a central reporting pipeline ([Handle errors](https://docs.flutter.dev/testing/errors), [PlatformDispatcher.onError](https://api.flutter.dev/flutter/dart-ui/PlatformDispatcher/onError.html)).
- Ensure logs are useful in DevTools (and in your production log sink) ([DevTools Logging view](https://docs.flutter.dev/tools/devtools/logging)).
- Add basic metrics around startup time, screen load time, and major network operations.
- Correlate network requests with user actions (trace IDs) where feasible.
- Add distributed tracing if the backend supports it and the product requires it.
- Use DevTools Performance and Logging views during development ([DevTools Performance view](https://docs.flutter.dev/tools/devtools/performance), [DevTools Logging view](https://docs.flutter.dev/tools/devtools/logging)).
- Don’t log PII, auth tokens, or secrets.
- **PR CHECK:** New error handling code MUST include (1) a user-facing error state and (2) a log/report event that excludes sensitive data.
- All interactive controls MUST be reachable by keyboard and expose accessible labels where needed ([User input & accessibility](https://docs.flutter.dev/ui/adaptive-responsive/input)).
- Use semantic widgets correctly (buttons are buttons, not gesture detectors on containers).
- Ensure focus order is logical and stable.
- Test with screen readers on mobile (VoiceOver/TalkBack) and keyboard-only navigation on desktop/web.
- Follow web accessibility guidance for Flutter web ([Web accessibility](https://docs.flutter.dev/platform-integration/web/web-accessibility)).
- Add custom Semantics nodes for complex controls.
- Use `FocusableActionDetector` for custom controls that need focus, mouse, and shortcuts ([User input & accessibility](https://docs.flutter.dev/ui/adaptive-responsive/input)).
- Don’t create custom clickable widgets that are not in the focus traversal order.
- **PR CHECK:** Any new custom interactive widget MUST include Semantics and be focusable with Tab.
- No user-facing strings may be hard-coded in widgets outside of localization files.
- Use Flutter’s localization tooling (`l10n.yaml`, ARB files, generated localizations) ([Internationalization](https://docs.flutter.dev/ui/internationalization)).
- Include pluralization and parameter typing in ARB placeholders.
- Ensure locale changes do not require app restart.
- Add per-locale formatting utilities for dates/numbers in a shared module.
- Enable `flutter: generate: true` and commit the ARB source files (not generated outputs) ([Internationalization](https://docs.flutter.dev/ui/internationalization)).
- Don’t concatenate strings for sentences (breaks translation).
- **PR CHECK:** Any new user-visible text in UI MUST be sourced from generated localizations (no raw string literals).
- CI MUST run: formatting, `flutter analyze`, unit/widget tests, and at least a smoke integration test.
- All shipped artifacts MUST be built in release mode ([Build modes](https://docs.flutter.dev/testing/build-modes)).
- Release processes MUST follow platform guidance for each target ([iOS release](https://docs.flutter.dev/deployment/ios), [Android release](https://docs.flutter.dev/deployment/android), [Web release](https://docs.flutter.dev/deployment/web), [Windows release](https://docs.flutter.dev/deployment/windows), [macOS release](https://docs.flutter.dev/deployment/macos), [Linux release](https://docs.flutter.dev/deployment/linux)).
- Lock the Flutter SDK version used by CI and developers (use a version manager if needed) to avoid “works on my machine” drift.
- Use obfuscation + split debug info when required, and store symbol files securely ([Obfuscation](https://docs.flutter.dev/deployment/obfuscate)).
- Automate store uploads and signing in CI (Codemagic/GitHub Actions/etc.) if security requirements are met.
- Validate web builds using a local server before deployment ([Web release](https://docs.flutter.dev/deployment/web)).
- Validate desktop packaging on a clean machine or VM.
- Don’t ship debug/profile builds to end users.
- **PR CHECK:** Any change to build configuration (Gradle/Xcode/web bootstrap) MUST include CI updates and a documented manual validation step.
- Perform network calls, database IO, file IO, or platform channel calls inside `build()`.
- Store auth tokens in `shared_preferences`.
- Scatter `kIsWeb` / `Platform.isX` checks across feature code.
- Hard-code visual styling (colors, typography, spacing, radii) outside `lib/design_system/**`.
- Import `dart:io` in code that is compiled for web.
- Introduce a second state management framework without an ADR and migration plan.
- Catch errors and ignore them (“empty catch”).
- Use `print()` as production logging.
- Add dependencies that do not support all target platforms without a documented fallback.
- Build large dynamic lists with `ListView(children: ...)` instead of virtualization.
- Use golden tests as the primary UI validation for cross-platform screens.
- Hard-code strings in UI widgets.
- Use mutable shared singletons as “global state”.
- Implement custom cryptography.
- Bypass TLS validation.
- **PR CHECK:** If a PR introduces any of the MUST NOT items, it fails review unless the PR is explicitly a migration away from that anti-pattern.
- **F001** MUST target **iOS, Android, web, macOS, Windows, Linux** from a single Dart/Flutter codebase.
- **F009** MUST declare supported OS/browser baselines in the repo (README or `docs/support.md`) and keep them updated per release.
- **F023** MAY use a monorepo tool (for example, Melos) to manage multiple internal packages, if the repo actually has multiple packages.
- **F039** MUST use a **single** primary state management framework across the codebase (default: Riverpod). Any exception requires a documented decision in the repo (ADR).
- **F145** MUST treat caching as a repository concern with explicit TTL/invalidations.
- **F151** New endpoints MUST be integrated through a repository method that can be unit tested without Flutter widgets.
- **F173** Any new configuration value added to the repo MUST be classified as public or secret. Secrets MUST NOT be committed and MUST be injected via CI/runtime.
- F001: MUST target **iOS, Android, web, macOS, Windows, Linux** from a single Dart/Flutter project.
- F002: MUST treat Flutter as the UI runtime for all platforms (no “rewrite per platform” strategy).
- F003: SHOULD use a single “product app” with internal packages/modules, not multiple divergent apps.
- F004: MAY maintain small platform-specific shims (native host code, JS glue) when required by platform capabilities.
- F005: MUST define explicit boundaries between UI, state, data, and platform integration.
- F006: MUST optimize for maintainability and predictable ownership.
- F007: MUST NOT treat “cross-platform” as “identical UI everywhere”. Platform-adaptive behavior is expected.
- F008: Any new platform-specific behavior MUST be implemented behind a platform adapter (not in feature UI code).
- F009: MUST declare supported OS/browser baselines in the project (README or `docs/support.md`) and keep them updated per release.
- F010: MUST run CI tests on at least one representative target per platform family: mobile, web, desktop.
- F011: MUST assume **Impeller is the default renderer** on iOS and on Android API 29+ in modern Flutter, so rendering behavior can differ from Skia ([Impeller](https://docs.flutter.dev/perf/impeller)).
- F012: SHOULD assume web builds may be produced in **default** or **WebAssembly** build modes ([Web renderers](https://docs.flutter.dev/platform-integration/web/renderers)).
- F013: SHOULD treat web as a hostile client: anything shipped to web is inspectable.
- F014: MAY explicitly support multiple web renderers when business requirements demand it (for example, fallback if Wasm not supported).
- F015: MUST make platform capability checks explicit and centralized.
- F016: MUST NOT assume a plugin works on all platforms without verifying.
- F017: Any new dependency on OS features (filesystem, biometrics, clipboard, notifications, window management) MUST include a platform support matrix (iOS/Android/web/macOS/Windows/Linux) in the PR description.
- F018: MUST use a **feature-first** structure for product code. Each feature is self-contained and owns its UI + state + data adapters.
- F019: MUST place shared cross-cutting code in **explicit** shared packages/modules (`core`, `design_system`, `platform`, `analytics`, etc.), not in random `utils/`.
- F020: MUST keep platform host directories (`android/`, `ios/`, `web/`, `macos/`, `windows/`, `linux/`) free of business logic. Only platform bootstrapping and configuration belongs there.
- F021: SHOULD use internal Dart packages (monorepo style) when the app exceeds a small team, or when features need hard boundaries.
- F022: SHOULD keep test directories co-located with the code they test (feature-oriented tests).
- F023: MAY use a monorepo tool (for example, Melos) to manage multiple internal packages, if the project actually has multiple packages.
- F024: MUST make every feature have a single entrypoint file (`feature.dart`) exporting public types.
- F025: MUST enforce imports: features may depend on `core/`, `platform/`, and other feature public APIs only.
- F026: MUST NOT allow features to import private files from other features (no `../features/other_feature/src/...`).
- F027: A file under `lib/features/<A>/...` MUST NOT import a non-public path from `lib/features/<B>/...` (only import `<B>/feature.dart` or other explicitly exported public APIs).
- F028: MUST implement **three primary layers**: 1. **Presentation (UI)**: Widgets only render and forward user intent. 2. **State/Controller**: Owns view state, coordinates async, calls repositories. 3. **Data/Platform**: Repositories and platform adapters; no Flutter UI imports.
- F029: MUST keep all platform integration (platform channels, FFI, JS interop) behind **platform adapters** in `lib/platform/`.
- F030: MUST ensure dependency direction is inward: UI depends on state, state depends on repositories, repositories depend on low-level clients.
- F031: SHOULD use immutable domain models and explicit mapping between transport DTOs and domain models.
- F032: SHOULD use “command” or “use-case” style methods for user intent (for example `submit()`, `refresh()`, `toggleDone()`), not random mutations. This mirrors Flutter’s architecture guidance for testable state management ([Architecture concepts](https://docs.flutter.dev/app-architecture/concepts)).
- F033: MAY add a domain layer when complexity warrants (validation rules, business policies, cross-feature invariants).
- F034: MUST treat repositories as the **single source of truth** for data retrieval and caching policy.
- F035: MUST keep platform adapters narrow with a stable interface.
- F036: MUST NOT import Flutter UI libraries (`package:flutter/...`) in data layer code.
- F037: MUST NOT pass `BuildContext` into repositories or services.
- F038: No class in `lib/features/**/data/` may import `package:flutter/` or `dart:ui`.
- Default: Riverpod (current major line in 2025-2026) ([Riverpod](https://riverpod.dev/)) **Allowed alternative (by exception):** BLoC, only when a feature benefits from event-driven modeling and the team commits to consistent usage ([BLoC library](https://bloclibrary.dev/)).
- F039: MUST use a **single** primary state management framework across the project (default: Riverpod). Any exception requires a documented decision in the project (ADR).
- F040: MUST model async state as explicit `loading/data/error` and handle all three in UI.
- F041: MUST keep state “close” to its feature. No global “god provider” that exposes everything.
- F042: SHOULD treat state objects as immutable.
- F043: SHOULD keep side effects in controllers/viewmodels, not in UI widgets.
- F044: MAY use `setState` only for ephemeral, local UI state that does not affect app behavior outside the current widget subtree (for example, hover state, text field local toggle).
- F045: MUST use providers as both dependency injection and state wiring when using Riverpod.
- F046: MUST prefer typed, structured error types (not strings).
- F047: MUST NOT store `BuildContext` in state.
- F048: MUST NOT mutate collections in-place in state.
- F049: Any state that influences navigation, network calls, persistence, auth, or cross-screen behavior MUST NOT be implemented with `setState`.
- F050: MUST avoid doing expensive work in `build()` (no JSON parsing, no sorting large lists, no synchronous file IO).
- F051: MUST virtualize long lists and grids (`ListView.builder`, slivers). Never build unbounded child lists eagerly.
- F052: MUST use `const` constructors wherever possible. Flutter explicitly calls this out as a performance best practice ([Performance best practices](https://docs.flutter.dev/perf/best-practices)).
- F053: SHOULD prefer creating reusable UI pieces as widgets (`StatelessWidget`) instead of helper functions for better rebuild behavior ([Performance best practices](https://docs.flutter.dev/perf/best-practices)).
- F054: SHOULD keep widgets under roughly one screen’s worth of complexity. Refactor when a widget exceeds ~200 lines or contains unrelated responsibilities.
- F055: MAY use `RepaintBoundary` when profiling confirms repaint isolation benefits.
- F056: MUST keep UI pure: render state, dispatch intents.
- F057: MUST use adaptive layouts (constraints + input modality) rather than platform branching.
- F058: MUST NOT call platform channels from UI.
- F059: MUST NOT trigger network calls from `build()`.
- F060: Any `ListView` with potentially large item count MUST use a builder/sliver constructor (no `ListView(children: ...)` for dynamic content).
- F061: MUST maintain a single design system module (`lib/design_system/` or `packages/design_system/`) that owns: `ThemeData`, `ColorScheme`, `TextTheme`, `ThemeExtension` tokens, and shared components.
- F062: MUST in feature UI code (`lib/features/**/presentation/**`), do not introduce hard-coded visual values: No `Color(0x...)` literals No `TextStyle(...)` constructors for app typography No magic spacing numbers (for example `EdgeInsets.all(17)`) Use theme and design tokens only.
- F063: MUST define and ship both `theme` and `darkTheme`, and set `themeMode` explicitly on `MaterialApp.router` (default: `ThemeMode.system`).
- F064: MUST use `ColorScheme` roles as the source of truth for colors. Do not use deprecated roles (`background`, `onBackground`, etc.) and do not rely on `ThemeData.primaryColor` for new code.
- F065: MUST use `ThemeExtension` for non-Material tokens (spacing scale, radii, shadows, motion durations, semantic colors). Every `ThemeExtension` MUST implement `copyWith` and `lerp`, and MUST be registered via `ThemeData.extensions`.
- F066: MUST all reusable UI components (buttons, text fields, cards, dialogs, banners, toasts) MUST live in the design system module. Features MAY compose them, but MUST NOT fork them.
- F067: MUST any change to tokens or component styles MUST update both light and dark themes (and high-contrast if supported) and MUST include a visual verification note in the PR.
- F068: SHOULD generate base color schemes from a small set of brand seed colors (Material Theme Builder or `ColorScheme.fromSeed`) and keep the seed(s) in source control.
- F069: SHOULD use component themes in `ThemeData` (for example `filledButtonTheme`, `inputDecorationTheme`, `snackBarTheme`) instead of styling widgets ad hoc.
- F070: SHOULD maintain canonical design tokens in a tool-friendly format (recommended: Design Tokens Community Group JSON format) and generate Dart token code from it.
- F071: SHOULD use a predictable naming scheme for tokens: semantic first (for example `surface`, `onSurface`, `danger`, `success`, `spacingMd`) rather than raw palette names (for example `blue500`).
- F072: SHOULD for major UX changes, follow an iterative design loop (prototype, test with users, adjust) and capture findings and decisions in `docs/design/` (or equivalent).
- F073: MAY support Android 12+ dynamic color ("Material You") using `dynamic_color` if the product wants platform-personalized themes. If enabled, you MUST provide a deterministic fallback scheme and verify all `ColorScheme` roles used by the app are populated and meet contrast expectations.
- F074: MUST access styling through `Theme.of(context)` and `Theme.of(context).extension<AppTokens>()` (or a `BuildContext` helper) in feature code.
- F075: MUST keep typography responsive to text scaling by using `TextTheme` roles, not hard-coded font sizes.
- F076: MUST treat design updates as governed changes: include design ticket link and update the component inventory.
- F077: MUST NOT sprinkle `Theme(...)` overrides inside random subtrees to "fix" styling. Fix the design system instead.
- F078: MUST NOT use `Colors.*` or `TextStyle(...)` directly in feature UI code.
- F079: MUST NOT store design tokens as mutable globals or static singletons outside `ThemeData`.
- F080: Outside `lib/design_system/**` (and tests), the PR MUST NOT introduce `Color(0x` literals, `TextStyle(` constructors, or new spacing magic numbers.
- F081: MUST use the Router API (Navigator 2.0 family) through an approved routing package (default: `go_router`) ([go_router](https://pub.dev/packages/go_router)).
- F082: MUST make deep links first-class: every user-facing screen that can be reached from outside the app must have a stable route.
- F083: MUST for web, configure URL strategy intentionally (path vs hash) and configure server rewrites when using path URLs ([URL strategy](https://docs.flutter.dev/ui/navigation/url-strategies)).
- F084: SHOULD keep routing configuration centralized in `lib/app/router.dart`, with features registering routes via a constrained API.
- F085: SHOULD prefer typed route parameters or explicit parsing functions to avoid runtime errors.
- F086: MAY use nested navigation stacks per tab/section when UX requires it.
- F087: MUST call `usePathUrlStrategy()` before `runApp` if you choose path URLs on web ([URL strategy](https://docs.flutter.dev/ui/navigation/url-strategies)).
- F088: MUST update `<base href>` when hosting at a non-root path ([URL strategy](https://docs.flutter.dev/ui/navigation/url-strategies)).
- F089: MUST NOT push routes with raw string concatenation for params (encode and validate).
- F090: MUST NOT disable browser back/forward behavior on web.
- F091: Any new route added MUST include: (1) a named route entry, (2) parameter validation, and (3) a deep-link test case or documented manual test steps for web.
- F092: MUST avoid blocking the main isolate. No heavy CPU work (large JSON parsing, crypto, image processing) on the UI isolate.
- F093: MUST add cancellation/timeouts to network operations and long-running tasks.
- F094: MUST treat all async errors as data: they must be caught and surfaced as state, not swallowed.
- F095: SHOULD offload CPU-heavy work using isolates where supported and where it measurably improves responsiveness.
- F096: SHOULD prefer chunking work or moving it to the backend for web when isolate support or transferability limits apply.
- F097: MAY use background isolates for non-UI workloads when the data passed between isolates is simple and transferable.
- F098: MUST use `AsyncValue.guard` (or equivalent) to standardize error handling in controllers.
- F099: MUST use `package:flutter/foundation.dart` `compute()` only for pure functions and transferable data.
- F100: MUST NOT rely on isolates as a blanket solution for web without validating behavior in the chosen web build mode.
- F101: Any operation that can exceed ~8ms in worst case MUST NOT run synchronously in `build()`, gesture handlers, or animation callbacks.
- F102: MUST measure performance in **profile or release** mode, not debug, because debug mode is intentionally slow ([Build modes](https://docs.flutter.dev/testing/build-modes)).
- F103: MUST use DevTools Performance view to validate frame timing when introducing animations, large lists, or heavy screens ([DevTools Performance view](https://docs.flutter.dev/tools/devtools/performance)).
- F104: MUST keep app size under active control and review bundle changes for web ([Measuring app size](https://docs.flutter.dev/perf/app-size)).
- F105: SHOULD prefer `const` widgets and small widget subtrees for rebuild control ([Performance best practices](https://docs.flutter.dev/perf/best-practices)).
- F106: SHOULD use image resizing (server-side or client-side decode sizing) and avoid decoding full-resolution assets unnecessarily.
- F107: MAY use deferred loading/components when initial download size is a user-visible problem ([Deferred components](https://docs.flutter.dev/perf/deferred-components)).
- F108: MUST use `ListView.builder`/slivers, pagination, and caching.
- F109: MUST keep animations cheap: avoid unnecessary opacity layers and large repaints.
- F110: MUST NOT ship performance changes without at least one profiling screenshot or metric when the change is large.
- F111: PRs that add a new animation, large list, or image-heavy screen MUST include a note describing how performance was validated (DevTools trace, profile build, or equivalent).
- F112: MUST keep web-only code behind a platform adapter. No web imports in feature UI/state unless it is strictly presentational.
- F113: MUST configure URL strategy intentionally and configure hosting rewrites for deep links when using path URLs ([URL strategy](https://docs.flutter.dev/ui/navigation/url-strategies)).
- F114: MUST if customizing web startup behavior, do it via `flutter_bootstrap.js` in `web/` and document the reason ([Web initialization](https://docs.flutter.dev/platform-integration/web/initialization)).
- F115: SHOULD consider WebAssembly build mode when supported by dependencies, and ensure JS interop uses modern APIs (`dart:js_interop`, `package:web`) as required by Flutter’s WebAssembly guidance ([Web renderers](https://docs.flutter.dev/platform-integration/web/renderers)).
- F116: SHOULD treat service worker caching as a product decision. Understand update semantics and cache busting ([Web initialization](https://docs.flutter.dev/platform-integration/web/initialization), [Web loading speed blog](https://blog.flutter.dev/best-practices-for-optimizing-flutter-web-loading-speed-7cc0df14ce5c)).
- F117: MAY use deferred imports/deferred components to reduce initial JS payload ([Deferred components](https://docs.flutter.dev/perf/deferred-components)).
- F118: MUST validate web release builds via a local web server, not opening files from disk ([Build and release web](https://docs.flutter.dev/deployment/web)).
- F119: MUST profile web performance with the recommended tools ([Web performance](https://docs.flutter.dev/perf/web-performance)).
- F120: MUST NOT store long-lived auth tokens in browser localStorage unless you have no alternative and have a documented risk acceptance. Prefer secure cookie-based sessions when feasible (see OWASP session guidance).
- F121: MUST NOT assume a renderer: test with the chosen renderer and document it ([Web renderers](https://docs.flutter.dev/platform-integration/web/renderers)).
- F122: Any new web-only capability MUST have (1) a platform adapter in `lib/platform/` and (2) a documented hosting/config requirement if it needs headers, rewrites, or HTTPS.
- F123: MUST support keyboard navigation and focus traversal for all interactive UI on desktop (and web) ([User input & accessibility](https://docs.flutter.dev/ui/adaptive-responsive/input)).
- F124: MUST treat window size as dynamic: layouts MUST be responsive and not assume phone constraints.
- F125: MUST ensure all plugins used by desktop builds support the target desktop OS or provide a stub/fallback.
- F126: SHOULD implement app-wide shortcuts using Flutter’s Actions/Shortcuts system ([Actions and shortcuts](https://docs.flutter.dev/ui/interactivity/actions-and-shortcuts)).
- F127: SHOULD provide desktop-appropriate UI affordances (hover states, right-click context menus, scroll wheel behavior).
- F128: MAY use platform-native look-and-feel widget sets when product design calls for it ([Building macOS apps](https://docs.flutter.dev/platform-integration/macos/building)).
- F129: MUST build desktop release artifacts using `flutter build windows|macos|linux` ([Desktop support](https://docs.flutter.dev/platform-integration/desktop)).
- F130: MUST follow OS distribution requirements (notarization on macOS, packaging on Windows, bundle completeness on Linux) ([Building macOS apps](https://docs.flutter.dev/platform-integration/macos/building), [Building Windows apps](https://docs.flutter.dev/platform-integration/windows/building), [Build Linux apps](https://docs.flutter.dev/platform-integration/linux/building)).
- F131: MUST NOT ship desktop UI that requires a mouse for basic flows.
- F132: Any new interactive control MUST be reachable via keyboard (Tab traversal) and activatable via Enter/Space on desktop/web.
- F133: MUST handle app lifecycle events for any feature that depends on foreground/background state (audio, timers, auth refresh, camera).
- F134: MUST ensure navigation respects platform conventions (system back on Android, swipe back where applicable).
- F135: MUST use platform-appropriate secure storage for sensitive user secrets (see Security rules).
- F136: SHOULD validate store requirements early (permissions, privacy disclosures, entitlements) ([Release iOS](https://docs.flutter.dev/deployment/ios), [Release Android](https://docs.flutter.dev/deployment/android)).
- F137: SHOULD keep startup time low by deferring non-critical initialization.
- F138: MAY use Android dynamic feature modules for large apps (deferred components) when install size is a major issue ([Deferred components](https://docs.flutter.dev/perf/deferred-components)).
- F139: MUST use release builds for validation ([Build modes](https://docs.flutter.dev/testing/build-modes)).
- F140: MUST use TestFlight/internal testing tracks before full release.
- F141: MUST NOT request broad permissions without a user-visible feature and a documented reason.
- F142: Any new permission (camera, location, notifications, storage) MUST include (1) a user-facing explanation string, and (2) a documented justification in the PR.
- F143: MUST centralize HTTP client configuration (timeouts, headers, auth, retry policy) in one module (`core/networking`).
- F144: MUST avoid direct HTTP calls in widgets or UI state objects.
- F145: MUST treat caching as a project concern with explicit TTL/invalidations.
- F146: SHOULD use idempotent retry with backoff for safe operations only.
- F147: SHOULD design API clients so they can be mocked in unit tests.
- F148: MAY use a local cache (database or file cache) for offline and large data, if product needs it.
- F149: MUST use DevTools Network view when diagnosing networking performance ([DevTools Network view](https://docs.flutter.dev/tools/devtools/network)).
- F150: MUST NOT log full request/response bodies when they can include PII or secrets.
- F151: New endpoints MUST be integrated through a project method that can be unit tested without Flutter widgets.
- F152: MUST use key-value storage only for small, non-sensitive preferences.
- F153: MUST use secure storage for sensitive user secrets (tokens), with explicit platform constraints.
- F154: MUST abstract persistence behind repositories so UI does not depend on storage backend.
- F155: SHOULD use a real database for structured data and queries (instead of stuffing JSON blobs into preferences).
- F156: SHOULD design migration strategy for any persisted schema.
- F157: MAY use platform-specific storage for desktop integrations (file pickers, recent documents) behind a platform adapter.
- F158: MUST use `shared_preferences` only for preferences and small settings ([shared_preferences](https://pub.dev/packages/shared_preferences)).
- F159: MUST use `flutter_secure_storage` for secrets where appropriate, understanding web limitations (HTTPS requirement, WebCrypto behavior) ([flutter_secure_storage](https://github.com/mogol/flutter_secure_storage)).
- F160: MUST NOT assume secure storage on web provides the same protection as OS keychains.
- F161: No auth token or refresh token may be stored in `shared_preferences`.
- F162: MUST never embed server secrets or privileged API keys in the client. Assume web builds are public.
- F163: MUST ensure all network communication uses TLS and fails closed on invalid certificates (default platform behavior; do not bypass).
- F164: MUST centralize auth/session handling so token refresh, logout, and revocation are consistent.
- F165: SHOULD prefer cookie-based web sessions with secure attributes when feasible, following OWASP guidance.
- F166: SHOULD use code obfuscation and split debug info for release builds when the product requires it ([Obfuscate Dart code](https://docs.flutter.dev/deployment/obfuscate)).
- F167: SHOULD avoid custom cryptography. Use platform-provided primitives (WebCrypto, Keychain/Keystore) or vetted libraries.
- F168: MAY add platform-specific hardening (root/jailbreak signals, jailbreak detection) if required by the product threat model.
- F169: MUST capture and report uncaught errors using both framework and platform-level hooks ([Handle errors](https://docs.flutter.dev/testing/errors), [PlatformDispatcher.onError](https://api.flutter.dev/flutter/dart-ui/PlatformDispatcher/onError.html)).
- F170: MUST maintain a dependency vulnerability review process.
- F171: MUST NOT store long-lived tokens in web local storage without a risk acceptance (OWASP).
- F172: MUST NOT log secrets.
- F173: Any new configuration value added to the project MUST be classified as public or secret. Secrets MUST NOT be committed and MUST be injected via CI/runtime.
- F174: MUST unit test repositories, pure functions, and business rules without Flutter.
- F175: MUST widget test UI components that contain logic (conditional rendering, validation, error states).
- F176: MUST maintain at least one integration test per critical user journey.
- F177: SHOULD avoid golden tests for highly dynamic UI and for platform-dependent rendering differences. Use them for stable design-system components only.
- F178: SHOULD run tests in CI for every PR.
- F179: MAY use platform-specific integration tests when a feature is platform-specific (desktop file dialogs, web auth flows).
- F180: MUST prefer integration testing via the `integration_test` package and profile when needed ([Integration tests](https://docs.flutter.dev/testing/integration-tests)).
- F181: MUST keep tests hermetic: mock network and time.
- F182: MUST NOT rely on manual testing as the only validation for cross-platform behavior.
- F183: Any bug fix MUST include a regression test at the lowest practical layer (unit first, widget next, integration last).
- F184: MUST use structured logging (category/name + fields). No raw `print()` in production code.
- F185: MUST route uncaught framework and async errors to a central reporting pipeline ([Handle errors](https://docs.flutter.dev/testing/errors), [PlatformDispatcher.onError](https://api.flutter.dev/flutter/dart-ui/PlatformDispatcher/onError.html)).
- F186: MUST ensure logs are useful in DevTools (and in your production log sink) ([DevTools Logging view](https://docs.flutter.dev/tools/devtools/logging)).
- F187: SHOULD add basic metrics around startup time, screen load time, and major network operations.
- F188: SHOULD correlate network requests with user actions (trace IDs) where feasible.
- F189: MAY add distributed tracing if the backend supports it and the product requires it.
- F190: MUST use DevTools Performance and Logging views during development ([DevTools Performance view](https://docs.flutter.dev/tools/devtools/performance), [DevTools Logging view](https://docs.flutter.dev/tools/devtools/logging)).
- F191: MUST NOT log PII, auth tokens, or secrets.
- F192: New error handling code MUST include (1) a user-facing error state and (2) a log/report event that excludes sensitive data.
- F193: MUST all interactive controls MUST be reachable by keyboard and expose accessible labels where needed ([User input & accessibility](https://docs.flutter.dev/ui/adaptive-responsive/input)).
- F194: MUST use semantic widgets correctly (buttons are buttons, not gesture detectors on containers).
- F195: MUST ensure focus order is logical and stable.
- F196: SHOULD test with screen readers on mobile (VoiceOver/TalkBack) and keyboard-only navigation on desktop/web.
- F197: SHOULD follow web accessibility guidance for Flutter web ([Web accessibility](https://docs.flutter.dev/platform-integration/web/web-accessibility)).
- F198: MAY add custom Semantics nodes for complex controls.
- F199: MUST use `FocusableActionDetector` for custom controls that need focus, mouse, and shortcuts ([User input & accessibility](https://docs.flutter.dev/ui/adaptive-responsive/input)).
- F200: MUST NOT create custom clickable widgets that are not in the focus traversal order.
- F201: Any new custom interactive widget MUST include Semantics and be focusable with Tab.
- F202: MUST no user-facing strings may be hard-coded in widgets outside of localization files.
- F203: MUST use Flutter’s localization tooling (`l10n.yaml`, ARB files, generated localizations) ([Internationalization](https://docs.flutter.dev/ui/internationalization)).
- F204: SHOULD include pluralization and parameter typing in ARB placeholders.
- F205: SHOULD ensure locale changes do not require app restart.
- F206: MAY add per-locale formatting utilities for dates/numbers in a shared module.
- F207: MUST enable `flutter: generate: true` and commit the ARB source files (not generated outputs) ([Internationalization](https://docs.flutter.dev/ui/internationalization)).
- F208: MUST NOT concatenate strings for sentences (breaks translation).
- F209: Any new user-visible text in UI MUST be sourced from generated localizations (no raw string literals).
- F210: MUST CI MUST run: formatting, `flutter analyze`, unit/widget tests, and at least a smoke integration test.
- F211: MUST all shipped artifacts MUST be built in release mode ([Build modes](https://docs.flutter.dev/testing/build-modes)).
- F212: MUST release processes MUST follow platform guidance for each target ([iOS release](https://docs.flutter.dev/deployment/ios), [Android release](https://docs.flutter.dev/deployment/android), [Web release](https://docs.flutter.dev/deployment/web), [Windows release](https://docs.flutter.dev/deployment/windows), [macOS release](https://docs.flutter.dev/deployment/macos), [Linux release](https://docs.flutter.dev/deployment/linux)).
- F213: SHOULD lock the Flutter SDK version used by CI and developers (use a version manager if needed) to avoid “works on my machine” drift.
- F214: SHOULD use obfuscation + split debug info when required, and store symbol files securely ([Obfuscation](https://docs.flutter.dev/deployment/obfuscate)).
- F215: MAY automate store uploads and signing in CI (Codemagic/GitHub Actions/etc.) if security requirements are met.
- F216: MUST validate web builds using a local server before deployment ([Web release](https://docs.flutter.dev/deployment/web)).
- F217: MUST validate desktop packaging on a clean machine or VM.
- F218: MUST NOT ship debug/profile builds to end users.
- F219: Any change to build configuration (Gradle/Xcode/web bootstrap) MUST include CI updates and a documented manual validation step.
- F220: MUST NOT perform network calls, database IO, file IO, or platform channel calls inside `build()`.
- F221: MUST NOT store auth tokens in `shared_preferences`.
- F222: MUST NOT scatter `kIsWeb` / `Platform.isX` checks across feature code.
- F223: MUST NOT hard-code visual styling (colors, typography, spacing, radii) outside `lib/design_system/**`.
- F224: MUST NOT import `dart:io` in code that is compiled for web.
- F225: MUST NOT introduce a second state management framework without an ADR and migration plan.
- F226: MUST NOT catch errors and ignore them (“empty catch”).
- F227: MUST NOT use `print()` as production logging.
- F228: MUST NOT add dependencies that do not support all target platforms without a documented fallback.
- F229: MUST NOT build large dynamic lists with `ListView(children: ...)` instead of virtualization.
- F230: SHOULD NOT use golden tests as the primary UI validation for cross-platform screens.
- F231: SHOULD NOT hard-code strings in UI widgets.
- F232: SHOULD NOT use mutable shared singletons as “global state”.
- F233: MUST NOT implement custom cryptography.
- F234: MUST NOT bypass TLS validation.
- F235: If a PR introduces any of the MUST NOT items, it fails review unless the PR is explicitly a migration away from that anti-pattern.
