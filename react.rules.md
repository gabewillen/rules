---
trigger: always_on
---

# React Architecture & Best Practices

## 1. Core & Architecture
- **React 19+ Semantics**: Target React 19.2 and React Compiler compatibility. Rely on the compiler instead of manual memoization (`useMemo`, `useCallback`). Use `StrictMode` in development.
- **Modern Tooling**: Use modern frameworks (Next, Remix, Vite) over Create React App.
- **UI Stack**: Treat `shadcn/ui` as owned source code. Assume a Tailwind v4 + CSS variable theming stack. Prefer native HTML semantics over custom role-based widgets. Avoid stacking multiple third-party UI kits.
- **Code Sharing**: Stay single-package until bundle isolation or independent release cycles are strictly required. Extract cross-app reusable primitives to an internal package or registry rather than hand-copying them.

## 2. Component Design
- **Function Components Only**: Never use class components.
- **Purity**: Components and hooks must be strictly pure and idempotent. Do not mutate props, state, or captured objects during render. No side effects in render.
- **Composition > Configuration**: Prefer explicit props, slots, `asChild`, and `children` over inheritance, HOCs, or massive configuration objects. Avoid `cloneElement` and `Children` outside of low-level infrastructure.
- **Prop Semantics**: Avoid boolean matrices for styles; use constrained union props (e.g., `variant`, `tone`). Use semantic prop names instead of generic ones like `data` or `config`.
- **Modern Refs**: Pass `ref` as a standard prop (React 19+). Do not use `forwardRef`. Use `useImperativeHandle` strictly for imperative contracts (focus, scroll, measure). Never use refs for render-driving state.

## 3. State Management
- **Local First**: Default to local state. Escalate to global stores only when local state is proven insufficient.
- **No Derived State**: Derive values from props or existing state during render instead of syncing them into state variables.
- **Immutability**: Always update arrays and objects immutably.
- **Explicit State Machines**: Use reducers or discriminated unions for multi-step flows and async UI. Use a single `status` field (`idle`, `loading`, `success`, `error`) instead of contradictory booleans (`isLoading`, `isSaving`).
- **State Resets**: Change a component's `key` prop to completely reset its state. Never use an effect to reset state when props change.
- **Context & Stores**: Use Context only for stable, globally shared values (auth, theme, locale). For rapidly changing external data, use `useSyncExternalStore`.

## 4. Effects (`useEffect`)
- **Strict Limits**: Use `useEffect` ONLY for synchronizing with external systems (DOM APIs, timers, third-party subscriptions).
- **No Logic in Effects**: Move user-driven logic to event handlers. Move pure data transformations to render. Do not use effects for data flow.
- **No Dependency Suppression**: Never disable `exhaustive-deps`. Design around the linter. Use `useEffectEvent` to read the latest state/props without triggering an effect re-run.

## 5. Directory Structure & Boundaries
- **`components/ui/`**: Low-level, generic primitives (shadcn, wrappers). Must not contain domain logic, network knowledge, or import from feature modules.
- **`features/<domain>/`**: Domain-specific UI and state. Must not import from other features' internals. Do not hide network/storage work inside leaf components.
- **`lib/` & `services/`**: Pure boundary code (HTTP, storage, auth, formatters).
- **`app/` & `routes/`**: Top-level modules that compose features and host route-level boundaries.
- **No Junk Drawers**: Never use "common", "misc", or "shared" folders for mixed concerns. Move shared code upward only if used by multiple call sites and it remains conceptually general.
- **Server/Client Isolation**: Server-only and client-only code must not share files unless explicitly supported by the framework.