# TypeScript Governance Rules

This document outlines an optimized, enforceable ruleset for TypeScript development across all runtime environments.

## 1. Type System & Soundness
- **Mandatory Strictness:** Enable `strict: true`. Require `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `useUnknownInCatchVariables`, `noPropertyAccessFromIndexSignature`, and `noImplicitOverride`.
- **Zero `any`:** Forbid explicit `any`. Use `unknown` and runtime narrowing. Isolate unavoidable interop `any` usage to boundary modules.
- **Safe Assertions:** Ban unsafe type assertions (`as T`). Use runtime validators or `satisfies`.
- **Null Safety:** Enable `strictNullChecks`. Handle `undefined` properly from index accesses. Prohibit non-null assertions (`!`).
- **Exhaustiveness:** Mandate exhaustive `switch` statements over discriminated unions using a `never`-based helper (e.g., `assertNever`).

## 2. Modules & Interop
- **ESM First:** Default to `"type": "module"` for new packages. Use `.mjs`/`.cjs` for explicit format markers. Never mix ESM and CJS semantics in the same file.
- **Resolution:** Set `module` and `moduleResolution` to `NodeNext` or `node20` for Node environments. Include explicit file extensions in relative imports.
- **Imports/Exports:** Prefer named exports. Use `verbatimModuleSyntax: true` to enforce `import type` correctness. Group imports consistently.

## 3. Architecture & Boundaries
- **Public API:** Strictly define public entrypoints via `package.json#exports`. Prohibit deep imports across package boundaries and internal implementation leakage.
- **Layering:** Keep cross-layer imports one-directional to prevent cycles. Use TypeScript project references (`composite: true`) in monorepos.
- **Runtime Isolation:** Provide dedicated tsconfigs with exact `lib` configurations for distinct runtimes (Node, DOM, edge). Never import Node-specific APIs in browser code.
- **Build Output:** Maintain strict separation between `rootDir` (source) and `outDir` (build). Never emit build artifacts into source directories.

## 4. Async & Concurrency
- **Async/Await:** Prefer `async/await` over `.then()` chains. Never use `async` callbacks inside `Array.prototype.forEach` (use `for...of` or `Promise.all`).
- **Promise Safety:** Forbid floating or unhandled promises; explicitly `await` them or mark them handled with `void`.
- **Concurrency Strategy:** Use `Promise.all()` for fail-fast independent tasks and `Promise.allSettled()` for robust batching.
- **Cancellation:** Manage long-running tasks using `AbortController` and `AbortSignal`.

## 5. Error Handling & Validation
- **Typed Errors:** Throw only `Error` subclasses (never primitives). Treat catch variables as `unknown`.
- **Resilience:** Do not silently swallow errors. Rethrow, wrap with context, or return explicit `Result` types. Ensure async entrypoints have robust, top-level boundaries.
- **Boundary Validation:** Treat all external data (network, env vars, IPC) as `unknown`. Validate at the boundary using schema validators (e.g., Zod) and derive static types directly from these schemas.

## 6. Toolchain, Testing & CI
- **Separation of Concerns:** Fast transpilers (esbuild, SWC) do not type-check. Always run a dedicated `tsc --noEmit` step.
- **Type-Aware Linting:** Run ESLint utilizing `typescript-eslint` with type-aware rules (e.g., `strict-type-checked`).
- **Testing Constraints:** Tests must be completely deterministic. Abstract external side-effects (network, filesystem) behind adapters and constrain mocking to module boundaries.
- **CI Pipelines:** CI must sequentially enforce: clean lockfile installation, auto-formatting check, linting, typechecking, tests, and build. Fail unequivocally on any type error.

## 7. Security & Performance
- **Supply-Chain:** Pin TypeScript as a dev dependency, enforce a single lockfile, and run automated vulnerability scans. Ban unreviewed `postinstall` scripts.
- **Safe Execution:** Never use dynamic code evaluation (`eval`, `new Function`) on untrusted inputs.
- **Type Costs:** Avoid massive union types and deeply nested conditional generics that degrade IDE responsiveness and compiler speed.

## 8. Banned Anti-Patterns
- `// @ts-ignore` or `// @ts-nocheck` (except in isolated generated code or as a temporary, ticketed workaround).
- Code that compiles solely due to `as any` or broad `eslint-disable` sweeps.
- Disabling `strict` mode in handwritten code.
- Setting `skipLibCheck: true` in published library configurations.
