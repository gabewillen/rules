---
trigger: always_on
---

# React Architecture & Best Practices

- REACT-001: MUST target React 19.2 and React Compiler compatibility. Rely on the compiler instead of manual memoization (`useMemo`, `useCallback`). Use `StrictMode` in development.
- REACT-002: MUST use modern frameworks (Next, Remix, Vite) over Create React App.
- REACT-003: MUST treat `shadcn/ui` as owned source code. Assume a Tailwind v4 + CSS variable theming stack. Prefer native HTML semantics over custom role-based widgets. Avoid stacking multiple third-party UI kits.
- REACT-004: MUST stay single-package until bundle isolation or independent release cycles are strictly required. Extract cross-app reusable primitives to an internal package or registry rather than hand-copying them.
- REACT-005: MUST never use class components.
- REACT-006: MUST keep components and hooks strictly pure and idempotent. Do not mutate props, state, or captured objects during render. No side effects in render.
- REACT-007: MUST prefer explicit props, slots, `asChild`, and `children` over inheritance, HOCs, or massive configuration objects. Avoid `cloneElement` and `Children` outside of low-level infrastructure.
- REACT-008: MUST avoid boolean matrices for styles; use constrained union props (e.g., `variant`, `tone`). Use semantic prop names instead of generic ones like `data` or `config`.
- REACT-009: MUST pass `ref` as a standard prop (React 19+). Do not use `forwardRef`. Use `useImperativeHandle` strictly for imperative contracts (focus, scroll, measure). Never use refs for render-driving state.
- REACT-010: MUST default to local state. Escalate to global stores only when local state is proven insufficient.
- REACT-011: MUST derive values from props or existing state during render instead of syncing them into state variables.
- REACT-012: MUST always update arrays and objects immutably.
- REACT-013: MUST use reducers or discriminated unions for multi-step flows and async UI. Use a single `status` field (`idle`, `loading`, `success`, `error`) instead of contradictory booleans (`isLoading`, `isSaving`).
- REACT-014: MUST change a component's `key` prop to completely reset its state. Never use an effect to reset state when props change.
- REACT-015: MUST use Context only for stable, globally shared values (auth, theme, locale). For rapidly changing external data, use `useSyncExternalStore`.
- REACT-016: MUST use `useEffect` ONLY for synchronizing with external systems (DOM APIs, timers, third-party subscriptions).
- REACT-017: MUST move user-driven logic to event handlers. Move pure data transformations to render. Do not use effects for data flow.
- REACT-018: MUST never disable `exhaustive-deps`. Design around the linter. Use `useEffectEvent` to read the latest state/props without triggering an effect re-run.
- REACT-019: MUST keep `components/ui/` for low-level, generic primitives (shadcn, wrappers). These must not contain domain logic, network knowledge, or import from feature modules.
- REACT-020: MUST keep `features/<domain>/` for domain-specific UI and state. Must not import from other features' internals. Do not hide network/storage work inside leaf components.
- REACT-021: MUST keep `lib/` and `services/` for pure boundary code (HTTP, storage, auth, formatters).
- REACT-022: MUST keep `app/` and `routes/` for top-level modules that compose features and host route-level boundaries.
- REACT-023: MUST never use "common", "misc", or "shared" folders for mixed concerns. Move shared code upward only if used by multiple call sites and it remains conceptually general.
- REACT-024: MUST ensure server-only and client-only code do not share files unless explicitly supported by the framework.