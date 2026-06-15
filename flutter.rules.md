# Flutter Coding Rules

These rules provide a highly optimized, enforceable standard for Flutter development. They apply to all code across iOS, Android, Web, macOS, Windows, and Linux targets.

## 1. Project Scope & Targets
- **Single Codebase:** Target all supported platforms from one unified codebase. Use Flutter as the sole UI runtime; avoid per-platform rewrites.
- **Platform Adaptation:** Expect adaptive behavior, not identical UI everywhere. Isolate platform-specific capabilities (native host code, JS glue) behind explicit platform adapters.
- **Centralized Checks:** Never assume a plugin works everywhere without verifying. Document OS/browser baselines and use a platform support matrix for new capabilities.
- **Rendering & Environments:** Assume Impeller is the default renderer on modern iOS/Android. Treat Web as a hostile client and plan for both default and WebAssembly build modes.

## 2. Architecture & Layout
- **Feature-First Structure:** Group code by feature (`presentation/`, `state/`, `data/`, `domain/`), not by technical type (e.g., no dumping grounds like `utils/`).
- **Strict Layering:**
  1. **Presentation (UI):** Render state and forward intents. No business logic.
  2. **State/Controller:** Manage view state, coordinate async tasks, call repositories.
  3. **Data/Platform:** Repositories and adapters. **Never** import `package:flutter/...` or `dart:ui` here.
- **Dependency Flow:** Inward only. UI → State → Data → Low-level clients.
- **Encapsulation:** Features must expose a single `feature.dart` entrypoint. Never import private files across feature boundaries.

## 3. State Management
- **Single Framework:** Universally adopt one primary state management framework (default: Riverpod).
- **Scoped State:** Keep state as close to the feature as possible. Avoid global "god providers".
- **Local State:** Use `setState` solely for local, ephemeral UI states (e.g., hover effects, text field toggles) that do not affect cross-screen behavior.
- **Async Modeling:** Explicitly model async state (`loading`/`data`/`error`). Handle all three states gracefully in the UI.
- **Immutability:** Treat state objects and collections as strictly immutable.

## 4. UI Composition & Design System
- **Widget Efficiency:** Heavily utilize `const` constructors. Keep `build()` fast—no synchronous I/O, heavy parsing, or network calls.
- **Virtualization:** Always virtualize dynamic or large lists (`ListView.builder`, slivers). Never use eager lists for unbounded children.
- **Widget Purity:** Prefer extracting reusable UI pieces into small `StatelessWidget`s (< 200 lines) rather than using helper functions.
- **Design System First:** Centralize all styling inside a `design_system` module utilizing `ThemeData`, `ColorScheme`, and `ThemeExtension`.
- **No Hardcoding:** Never use magic numbers (`EdgeInsets.all(17)`), `Color(0x...)`, or inline `TextStyle(...)` inside feature UI. Consume tokens strictly from the design system.

## 5. Navigation & Routing
- **Router API:** Use Navigator 2.0 via a designated routing package (default: `go_router`).
- **Deep Linking:** Treat deep links as first-class citizens. Ensure all user-facing screens have stable, addressable routes. Use typed route parameters.
- **Web Navigation:** Configure the URL strategy intentionally (path vs. hash). Do not disable default browser back/forward behaviors.

## 6. Concurrency & Performance
- **Protect the UI Thread:** Never block the main isolate. Offload heavy CPU work (JSON parsing, cryptography) to isolates using `compute()` where supported.
- **Async Guardrails:** Implement timeouts and cancellation for network and long-running tasks. Catch all async errors; do not swallow them.
- **Profiling:** Measure and validate performance in **profile or release mode** only. Use DevTools to trace animations and heavy lists.
- **Animations:** Keep animations cheap by minimizing opacity layers and avoiding large repaints. Use `RepaintBoundary` when profiling confirms its benefit.

## 7. Platform-Specific Constraints
- **Web:** Keep web-only code behind an adapter. Validate web release builds via a local web server. Avoid storing long-lived auth tokens in `localStorage`.
- **Desktop:** Support comprehensive keyboard navigation, focus traversal, and dynamic window sizing. Never assume mobile constraints or require a mouse for fundamental flows.
- **Mobile:** Handle app lifecycle events explicitly (backgrounding, audio). Respect platform navigation conventions (Android system back, iOS swipe back).

## 8. Data, Networking & Security
- **Networking:** Centralize HTTP client configuration (timeouts, headers, auth, retries). Do not make direct HTTP calls from widgets.
- **Storage:** Use `shared_preferences` **only** for small, non-sensitive preferences. Use secure storage for tokens. Abstract all persistence behind repositories.
- **Security:** Never embed secrets or API keys in the client. Enforce TLS validation. Rely on vetted, platform-provided cryptographic primitives.

## 9. Testing & Observability
- **Testing Pyramid:** Unit test repositories and pure logic without Flutter. Widget test UI logic. Maintain integration tests for critical user journeys. Keep tests hermetic.
- **Observability:** Use structured logging. Centralize reporting for uncaught errors. Do not log PII, auth tokens, or secrets.

## 10. Accessibility & Internationalization
- **Accessibility (a11y):** All interactive controls must be keyboard reachable, maintain a logical focus order, and expose semantic labels. Do not use gesture detectors on containers as buttons.
- **Internationalization (i18n):** Never hard-code user-facing strings. Utilize Flutter's localization tooling (ARB files, `l10n.yaml`).

## 11. Strict Anti-Patterns
- **NEVER** perform network calls, database/file I/O, or platform channel calls inside `build()`.
- **NEVER** store auth tokens in `shared_preferences` or log sensitive secrets.
- **NEVER** scatter `kIsWeb` or `Platform.isX` checks across feature UI; abstract them into platform adapters.
- **NEVER** use empty `catch` blocks. All errors must be handled or explicitly logged.
- **NEVER** use raw `print()` for production logging.
- **NEVER** import `dart:io` in files that will be compiled for the web.
