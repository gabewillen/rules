# TypeScript Governance Rules

This document outlines an optimized, enforceable ruleset for TypeScript development across all runtime environments.

- TS-001: MUST enable `strict: true`. Require `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `useUnknownInCatchVariables`, `noPropertyAccessFromIndexSignature`, and `noImplicitOverride`.
- TS-002: MUST forbid explicit `any`. Use `unknown` and runtime narrowing. Isolate unavoidable interop `any` usage to boundary modules.
- TS-003: MUST ban unsafe type assertions (`as T`). Use runtime validators or `satisfies`.
- TS-004: MUST enable `strictNullChecks`. Handle `undefined` properly from index accesses. Prohibit non-null assertions (`!`).
- TS-005: MUST mandate exhaustive `switch` statements over discriminated unions using a `never`-based helper (e.g., `assertNever`).
- TS-006: MUST default to `"type": "module"` for new packages. Use `.mjs`/`.cjs` for explicit format markers. Never mix ESM and CJS semantics in the same file.
- TS-007: MUST set `module` and `moduleResolution` to `NodeNext` or `node20` for Node environments. Include explicit file extensions in relative imports.
- TS-008: MUST prefer named exports. Use `verbatimModuleSyntax: true` to enforce `import type` correctness. Group imports consistently.
- TS-009: MUST strictly define public entrypoints via `package.json#exports`. Prohibit deep imports across package boundaries and internal implementation leakage.
- TS-010: MUST keep cross-layer imports one-directional to prevent cycles. Use TypeScript project references (`composite: true`) in monorepos.
- TS-011: MUST provide dedicated tsconfigs with exact `lib` configurations for distinct runtimes (Node, DOM, edge). Never import Node-specific APIs in browser code.
- TS-012: MUST maintain strict separation between `rootDir` (source) and `outDir` (build). Never emit build artifacts into source directories.
- TS-013: MUST prefer `async/await` over `.then()` chains. Never use `async` callbacks inside `Array.prototype.forEach` (use `for...of` or `Promise.all`).
- TS-014: MUST forbid floating or unhandled promises; explicitly `await` them or mark them handled with `void`.
- TS-015: MUST use `Promise.all()` for fail-fast independent tasks and `Promise.allSettled()` for robust batching.
- TS-016: MUST manage long-running tasks using `AbortController` and `AbortSignal`.
- TS-017: MUST throw only `Error` subclasses (never primitives). Treat catch variables as `unknown`.
- TS-018: MUST NOT silently swallow errors. Rethrow, wrap with context, or return explicit `Result` types. Ensure async entrypoints have robust, top-level boundaries.
- TS-019: MUST treat all external data (network, env vars, IPC) as `unknown`. Validate at the boundary using schema validators (e.g., Zod) and derive static types directly from these schemas.
- TS-020: MUST ensure fast transpilers (esbuild, SWC) do not type-check. Always run a dedicated `tsc --noEmit` step.
- TS-021: MUST run ESLint utilizing `typescript-eslint` with type-aware rules (e.g., `strict-type-checked`).
- TS-022: MUST ensure tests are completely deterministic. Abstract external side-effects (network, filesystem) behind adapters and constrain mocking to module boundaries.
- TS-023: MUST sequentially enforce in CI: clean lockfile installation, auto-formatting check, linting, typechecking, tests, and build. Fail unequivocally on any type error.
- TS-024: MUST pin TypeScript as a dev dependency, enforce a single lockfile, and run automated vulnerability scans. Ban unreviewed `postinstall` scripts.
- TS-025: MUST NOT use dynamic code evaluation (`eval`, `new Function`) on untrusted inputs.
- TS-026: MUST avoid massive union types and deeply nested conditional generics that degrade IDE responsiveness and compiler speed.
- TS-027: MUST NOT use `// @ts-ignore` or `// @ts-nocheck` (except in isolated generated code or as a temporary, ticketed workaround).
- TS-028: MUST NOT allow code that compiles solely due to `as any` or broad `eslint-disable` sweeps.
- TS-029: MUST NOT disable `strict` mode in handwritten code.
- TS-030: MUST NOT set `skipLibCheck: true` in published library configurations.
