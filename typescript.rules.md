---
alwaysApply: true
---

# TypeScript Governance Rules (2025-2026)

This document is an opinionated, enforceable ruleset for AI coding agents working in TypeScript 5.x+ codebases.

## 1. Scope and assumptions

This ruleset governs TypeScript code authored or maintained by AI agents in Node, browser, serverless/edge, libraries, and full-stack repos. Rules use RFC 2119 keywords.

* **TSR-001:** Code changes MUST keep the repository type-checking clean under the repository’s enforced TypeScript version (5.x+) and tsconfig(s) in CI.
* **TSR-002:** New TypeScript code MUST be written to pass with `strict: true` and the additional strictness flags mandated in this document (see tsconfig discipline). \[S1]
* **TSR-003:** All externally obtained data (network, disk, env vars, user input, IPC, database) MUST be treated as `unknown` at the boundary and validated or parsed before use. \[S38]
* **TSR-004:** Each distinct runtime environment (Node, DOM, WebWorker, edge) MUST have a dedicated tsconfig with an appropriate `lib` and module settings, connected via project references when in one repo. \[S12]
* **TSR-005:** AI agents MUST NOT introduce new `any` types except where explicitly permitted by this ruleset (see Type system rules).
* **TSR-006:** Generated code MUST be isolated (separate folder and tsconfig or excluded) and MUST NOT block CI quality gates for handwritten code.

## 2. Toolchain and versions (TypeScript, Node, package manager expectations)

Tooling MUST be pinned and reproducible across machines and CI.

* **TSR-007:** Repositories MUST pin TypeScript as a devDependency and MUST NOT rely on a globally installed `tsc`.
* **TSR-008:** Repositories MUST specify supported Node.js versions via `package.json#engines.node` and MUST align CI to those versions. Default baseline SHOULD be Node 24 LTS or newer. \[S21]
* **TSR-009:** Repositories MUST use a single, workspace-capable package manager across the repo (npm, pnpm, or Yarn) and MUST commit exactly one lockfile (e.g., `package-lock.json`, `pnpm-lock.yaml`, or `yarn.lock`).
* **TSR-010:** CI installs MUST use the package manager’s clean, lockfile-driven install mode (e.g., `npm ci`) and MUST fail if the lockfile is out of sync. \[S31]
* **TSR-011:** Repos MUST record the chosen package manager and version using `package.json#packageManager` (Corepack) or equivalent, and CI MUST enforce it.
* **TSR-012:** For TypeScript language changes, repos SHOULD review TypeScript 5.x release notes when upgrading minors, and SHOULD track recent stable 5.x minors unless blocked by a documented compatibility constraint. \[S16]\[S17]

## 3. Project structure (monorepos, packages, layering, boundaries)

Structure MUST make boundaries enforceable by the typechecker, bundler, and runtime.

* **TSR-013:** Each publishable package MUST be independently buildable and testable, with its own `package.json`, `tsconfig.json`, and `src/` directory.
* **TSR-014:** Each package MUST treat its `exports` map as the public API surface. Importing another internal file path (deep import) across package boundaries MUST NOT be done. \[S20]
* **TSR-015:** Internal-only modules MUST reside under an explicit internal namespace (e.g., `src/internal/**`) and MUST NOT be exported from the package’s `exports` map. \[S20]
* **TSR-016:** Cross-layer imports MUST be one-directional (e.g., `app` -> `domain` -> `data`), and cycles across packages or layers MUST NOT be introduced.
* **TSR-017:** Each package MUST have exactly one source root (`rootDir`) and one output root (`outDir`) and MUST NOT emit build artifacts into `src/`.
* **TSR-018:** Monorepos SHOULD use TypeScript project references between packages to enforce layering and to accelerate builds. \[S13]
* **TSR-019:** Workspace root MUST contain a build orchestrator entry (scripts or task runner) that can run: format check, lint, typecheck, tests, and build across all packages.

## 4. tsconfig discipline (strictness flags, project references, composite builds)

tsconfig choices are the primary governance surface for type safety and module correctness.

* **TSR-020:** Each package MUST have a `tsconfig.json` checked into version control (no implicit defaults). \[S1]
* **TSR-021:** `strict` MUST be enabled in all non-generated code tsconfigs. \[S1]
* **TSR-022:** The following strictness flags MUST be enabled in all non-generated code: `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `useUnknownInCatchVariables`, `noPropertyAccessFromIndexSignature`, `noImplicitOverride`. \[S2]\[S3]\[S4]\[S6]\[S5]
* **TSR-023:** `verbatimModuleSyntax: true` MUST be enabled for ESM projects to prevent type/value import confusion and to force `import type` correctness. \[S7]
* **TSR-024:** If any code is transpiled by a per-file tool (esbuild, swc, babel), `isolatedModules: true` MUST be enabled in the checked tsconfig used by editors and CI. \[S8]
* **TSR-025:** Each referenced project in a project-reference graph MUST set `composite: true`. \[S13]\[S14]
* **TSR-026:** Repos using project references MUST build via `tsc --build` (or equivalent orchestration that invokes it) and MUST NOT rely on ad-hoc `tsc` runs that ignore references. \[S13]
* **TSR-027:** All packages in a project-reference graph SHOULD enable `incremental: true` (or `tsc --build` default behavior) and commit no `.tsbuildinfo` files. \[S15]
* **TSR-028:** `skipLibCheck` MUST be `false` for libraries intended for reuse; it MAY be set to `true` for applications only with a documented performance rationale and a plan to periodically run a full `skipLibCheck: false` check in CI. \[S47]
* **TSR-029:** The repo MUST maintain separate tsconfigs for incompatible runtime libs (e.g., Node-only vs DOM-only code) and connect them with references if they share code. \[S12]
* **TSR-030:** Node-targeted ESM packages MUST use `module` modes that integrate with Node’s ESM behavior (e.g., `NodeNext` or `node20`) and MUST align with the package.json `type` and file extensions. \[S9]\[S19]
* **TSR-031:** Bundler-targeted packages (where bundler owns resolution) MAY use `moduleResolution: bundler`, but libraries emitting `.d.ts` SHOULD prefer `moduleResolution: NodeNext` to keep declaration imports compatible for consumers. \[S12]\[S11]\[S10]
* **TSR-032:** Type checking and emitting MUST be separable. CI MUST have a `tsc --noEmit` step even if the build uses a bundler. \[S32]\[S33]
* **TSR-033:** The repository MUST provide a strict baseline tsconfig. Example baseline:
  \\

```jsonc
{
  // Baseline tsconfig.json (copy/paste; adjust only with documented rationale)
  "compilerOptions": {
    /* Build graph */
    "composite": true,
    "incremental": true,
    /* Language + emit target */
    "target": "ES2022",
    "lib": ["ES2022"],
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    /* Output */
    "rootDir": "./src",
    "outDir": "./dist",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "noEmitOnError": true,
    /* Interop + runtime correctness */
    "verbatimModuleSyntax": true,
    "isolatedModules": true,
    "useDefineForClassFields": true,
    "forceConsistentCasingInFileNames": true,
    /* Strictness */
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "useUnknownInCatchVariables": true,
    "noPropertyAccessFromIndexSignature": true,
    "noImplicitOverride": true,
    /* Hygiene */
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    /* Optional: performance vs correctness tradeoff (see rules below) */
    "skipLibCheck": false
  },
  "include": ["src/**/*.ts"],
  "exclude": ["dist", "node_modules"]
}
```

(This example targets Node ESM. Adjust `lib`, `module`, and `moduleResolution` per-runtime.) \[S1]\[S2]\[S3]\[S4]\[S5]\[S6]\[S7]\[S8]\[S9]\[S10]\[S14]\[S15]\[S41]\[S46]\[S42]\[S45]\[S43]\[S44]

## 5. Formatting and style (Prettier or equivalent, naming, imports)

Formatting rules MUST be automatic and non-negotiable. Style rules aim to reduce diff noise and runtime ambiguity.

* **TSR-034:** The repo MUST use an auto-formatter (Prettier preferred) and MUST run it in CI in check mode. \[S27]\[S28]
* **TSR-035:** Formatting configuration MUST be repo-local (checked into version control); relying on developer-global editor settings MUST NOT be done. \[S27]
* **TSR-036:** Code MUST NOT be merged if it changes formatting without semantic changes (format first, then refactor).
* **TSR-037:** Exports intended for consumption by other modules SHOULD be named exports; default exports MAY be used only when a framework or platform requires them.
* **TSR-038:** Type-only imports MUST use `import type` (enforced via `verbatimModuleSyntax` and ESLint). \[S7]
* **TSR-039:** Imports MUST be grouped in this order: (1) built-ins, (2) external packages, (3) internal packages, (4) relative imports; each group separated by a blank line.
* **TSR-040:** File and directory names MUST be consistent across the repo (choose one: kebab-case or camelCase) and MUST match import casing exactly. \[S46]
* **TSR-041:** Public types MUST use `PascalCase`; values MUST use `camelCase`; constants MAY use `SCREAMING_SNAKE_CASE` only for true constants (no runtime mutation).

## 6. Static analysis and linting (ESLint + typescript-eslint, rule philosophy)

Linting MUST find real bugs, not stylistic bikeshedding. Prefer type-aware rules where they reduce runtime surprises.

* **TSR-042:** The repo MUST use ESLint for JavaScript/TypeScript linting and MUST run it in CI. \[S25]
* **TSR-043:** The repo MUST use typescript-eslint for TypeScript-aware linting (parser + plugin). \[S22]\[S23]
* **TSR-044:** CI linting MUST run with type information enabled for packages that produce runtime code (use a typed config such as `strict-type-checked`). \[S22]\[S23]
* **TSR-045:** ESLint configuration MUST use the supported config system for the repo (Flat Config preferred for new repos) and MUST be committed. \[S25]
* **TSR-046:** Typed linting in monorepos SHOULD use `parserOptions.projectService` to align ESLint type information with editor behavior and project references. \[S24]
* **TSR-047:** Rules that can produce false positives at scale MUST start at severity `warn` and MUST be promoted to `error` only once fixed repo-wide. \[S26]
* **TSR-048:** The lint configuration MUST disable conflicting core rules in favor of their TypeScript-aware equivalents (example: use `@typescript-eslint/no-unused-vars` instead of `no-unused-vars`). \[S23]
* **TSR-049:** The repo MUST provide an ESLint config example equivalent to the following (adjust file globs per repo):
  \\

```js
// eslint.config.mjs (Flat Config)
// Requires: eslint, typescript-eslint
import eslint from "@eslint/js";
import tseslint from "typescript-eslint";
import path from "node:path";
import { fileURLToPath } from "node:url";
const tsconfigRootDir = path.dirname(fileURLToPath(import.meta.url));
export default tseslint.config(
  eslint.configs.recommended,
  // Type-aware rule sets (slower, but catches real bugs)
  ...tseslint.configs.strictTypeChecked,
  ...tseslint.configs.stylisticTypeChecked,
  {
    files: ["**/*.ts", "**/*.tsx"],
    languageOptions: {
      parserOptions: {
        // Prefer projectService for monorepos and project references.
        projectService: true,
        tsconfigRootDir
      }
    },
    rules: {
      // Governance: keep runtime and types aligned
      "@typescript-eslint/consistent-type-imports": ["error", { "prefer": "type-imports" }],
      // Async correctness
      "@typescript-eslint/no-floating-promises": "error",
      "@typescript-eslint/no-misused-promises": "error",
      // Hygiene
      "@typescript-eslint/no-unused-vars": ["error", { "argsIgnorePattern": "^_", "varsIgnorePattern": "^_" }]
    }
  }
);
```

\[S22]\[S23]\[S24]\[S25]

* **TSR-050:** ESLint MUST NOT be used as a substitute for TypeScript type-checking; both MUST run (they detect different categories of issues).

## 7. Type system rules (strict mode, unknown vs any, generics, inference control)

Type system rules aim to keep static types aligned with runtime reality and to keep types maintainable at scale.

* **TSR-051:** New code MUST NOT introduce explicit `any` (including `any[]`, `Record<string, any>`, `as any`, or implicit anys). Use `unknown` and narrow instead. \[S1]
* **TSR-052:** If `any` is unavoidable (interop with untyped third-party), it MUST be confined to a single boundary module and MUST be converted to `unknown` immediately after ingestion.
* **TSR-053:** Type assertions (`as T`) MUST be preceded by a runtime check or schema parse that makes the assertion true, or the assertion MUST be replaced by a parser/validator.
* **TSR-054:** Generic type parameters MUST be constrained when possible (e.g., `T extends Foo`) and MUST NOT be left unconstrained when the implementation assumes structure.
* **TSR-055:** Public generics MUST be simple enough to understand from the signature alone; if the type requires multi-screen conditional gymnastics, it MUST be hidden behind a named helper type or refactored.
* **TSR-056:** Code SHOULD prefer `satisfies` for validating object literals against a type without widening the inferred type, and SHOULD avoid `as` for that use case.
* **TSR-057:** Mapped/conditional types in hot paths of the developer experience (build times, editor) SHOULD be minimized; prefer simpler, explicit types for exported APIs.

## 8. Nullability and soundness (strictNullChecks, narrowing, exhaustiveness)

Null and undefined are the dominant cause of runtime crashes. Treat them as first-class.

* **TSR-058:** `strictNullChecks` MUST be enabled (via `strict`). \[S1]
* **TSR-059:** `noUncheckedIndexedAccess` MUST be enabled and code MUST handle the resulting `undefined` in index access results via narrowing or defaults. \[S2]
* **TSR-060:** `exactOptionalPropertyTypes` MUST be enabled and code MUST NOT treat `prop?: T` as `T | undefined` unless `undefined` is explicitly part of `T`. \[S3]
* **TSR-061:** Non-null assertions (`expr!`) MUST NOT be used in production code except in a tiny, audited boundary where a runtime invariant is proven and documented.
* **TSR-062:** Switch statements over discriminated unions MUST be exhaustive. Use a `never`-based `assertNever(x)` helper and ensure the default branch is unreachable.
* **TSR-063:** Code SHOULD narrow unions using `typeof`, `instanceof`, `in`, and user-defined type guards before accessing properties on `unknown` or union types.

## 9. API design (public surface, internal modules, stability, deprecations)

Public APIs are forever. Optimize for stability, clarity, and safe evolution.

* **TSR-064:** Every package MUST declare a public API via `package.json#exports`; only exported entrypoints are considered supported. \[S20]\[S29]
* **TSR-065:** Public entrypoints MUST be minimal. Re-export only what is intended for consumers; avoid accidental exports via `export *` from internal modules.
* **TSR-066:** Public types MUST avoid leaking private implementation details (e.g., concrete classes, internal discriminants) unless part of the contract.
* **TSR-067:** Published packages MUST follow Semantic Versioning, and breaking changes MUST only ship in a major version bump. \[S39]
* **TSR-068:** Any planned breaking change MUST first be introduced as a deprecated API (`/** @deprecated */`) with a migration path, then removed in a major version. \[S39]
* **TSR-069:** If the repo uses TypeScript’s deprecation warnings, it MAY use `ignoreDeprecations` only as a temporary upgrade aid and MUST not silence deprecations indefinitely. \[S49]

## 10. Error handling (exceptions vs result types, never type usage, async errors)

Errors MUST be modeled intentionally. Default JavaScript behavior is surprising.

* **TSR-070:** Code MUST throw only `Error` (or subclasses). Throwing strings, numbers, or plain objects MUST NOT be done.
* **TSR-071:** Library code MUST NOT swallow errors; it MUST either (a) rethrow, (b) wrap with causal context, or (c) return an explicit `Result` type when the error is an expected business outcome.
* **TSR-072:** Application code MAY use exceptions for control flow only at process boundaries (e.g., request handler), and MUST convert them to appropriate responses/logs.
* **TSR-073:** Catch variables MUST be treated as `unknown` and narrowed before use (enforce via `useUnknownInCatchVariables`). \[S4]
* **TSR-074:** Error messages MUST include actionable context (operation, key identifiers) and MUST NOT leak secrets (tokens, credentials, PII).
* **TSR-075:** When using `never` for exhaustiveness, the code MUST ensure the `never` path is unreachable at runtime (use an `assertNever` that throws).
* **TSR-076:** Async entrypoints (HTTP handlers, jobs, event handlers) MUST have a top-level error boundary that logs and maps errors to a stable output.

## 11. Async and concurrency (Promises, async/await, cancellation, AbortController)

Async bugs are production bugs. Make concurrency explicit and cancellations plumbed end-to-end.

* **TSR-077:** Async functions MUST return `Promise<T>` (implicitly via `async`) and MUST NOT mix callback-style APIs without an explicit wrapper.
* **TSR-078:** New async code SHOULD use `async`/`await` for readability; chaining `.then()` MAY be used only for simple transformations where errors are still correctly propagated.
* **TSR-079:** Fire-and-forget Promises MUST be explicitly marked and handled: either `await` them, or use `void promise.catch(handle)`; leaving floating Promises MUST NOT be done (enforce via ESLint). \[S23]
* **TSR-080:** `Array.prototype.forEach` MUST NOT be used with `async` callbacks. Use `for...of` (sequential) or `Promise.all`/`Promise.allSettled` (concurrent).
* **TSR-081:** Concurrency MUST be explicit: use `Promise.all()` when tasks are independent and failure should fail-fast. \[S36]
* **TSR-082:** Code SHOULD use `Promise.allSettled()` when it needs results from all tasks regardless of individual failures, and MUST handle each outcome explicitly. \[S37]
* **TSR-083:** APIs that can be long-running or user-cancelable MUST accept an `AbortSignal` and MUST respect cancellation promptly. \[S35]
* **TSR-084:** New cancellation-capable code MUST use `AbortController`/`AbortSignal` (not custom boolean flags) unless a platform forbids it. \[S35]

## 12. Performance considerations (allocation, hot paths, structural typing costs)

Performance rules focus on preventing accidental regressions and pathological type-checking slowdowns.

* **TSR-085:** Any performance-motivated change MUST include a benchmark or measurement methodology in the PR description (before/after).
* **TSR-086:** Code in hot paths MUST avoid unnecessary allocations (e.g., avoid creating arrays/objects inside tight loops) unless measured and acceptable.
* **TSR-087:** Do not rely on V8 engine quirks. Performance-sensitive code MUST target correctness first, then measured optimization.
* **TSR-088:** Performance-sensitive code MUST NOT rely on JavaScript engine quirks; it MUST be correct first, then optimized based on measurement.
* **TSR-089:** Exported types SHOULD NOT use unions with hundreds of members; prefer data-driven runtime tables and simple types.

## 13. ESM vs CommonJS rules (module resolution, interop, exports discipline)

Most runtime surprises in 2025-2026 TypeScript come from mismatched ESM/CJS assumptions. These rules make module intent explicit.

* **TSR-090:** New Node packages SHOULD be ESM-first: set `package.json#type` to `"module"` and emit ESM, unless a documented consumer constraint requires CommonJS. \[S19]\[S29]
* **TSR-091:** Node packages MUST use Node’s explicit markers for module format: `.mjs`/`.cjs` extensions or `package.json#type`. Mixed, implicit behavior MUST NOT be relied on. \[S19]
* **TSR-092:** TypeScript `module` and `moduleResolution` MUST be set to a Node-integrated mode (`NodeNext` or `node20`) when targeting Node’s ESM/CJS behavior. \[S9]
* **TSR-093:** Relative imports in Node ESM output MUST be resolvable by Node at runtime. When emitting ESM, relative import specifiers SHOULD include file extensions as required by Node’s ESM resolver. \[S19]\[S9]
* **TSR-094:** Packages intended to support both `import` and `require` MUST use conditional exports (`exports` map) and MUST test both entrypoints in CI. \[S20]
* **TSR-095:** Interop with CommonJS MUST follow TypeScript’s documented ESM/CJS interoperability guidance; do not assume default import behavior without verifying runtime semantics. \[S40]
* **TSR-096:** The `exports` map MUST NOT expose deep internal paths; only explicit entrypoints are allowed. \[S20]
* **TSR-097:** Internal path aliases for in-package imports MAY use the `imports` field with `#`-prefixed specifiers; external-facing aliases MUST NOT use `imports`. \[S20]

## 14. Runtime boundaries (Node vs browser vs edge runtimes)

TypeScript types do not enforce runtime availability. Boundaries must be explicit in code and packaging.

* **TSR-098:** Each runtime target MUST have its own tsconfig with correct `lib` and (when relevant) `types`, and those tsconfigs MUST be used by both editor tooling and CI. \[S12]
* **TSR-099:** Browser-targeted code MUST NOT import Node-only built-ins (fs, path, crypto Node APIs) unless behind a runtime-guarded, conditionally exported module.
* **TSR-100:** Edge/serverless code MUST avoid Node APIs not supported by the target platform; platform-specific shims MUST be isolated behind adapter interfaces.
* **TSR-101:** Multi-runtime libraries MUST provide separate entrypoints per runtime using conditional exports (e.g., `browser`, `node`) and MUST test each entrypoint. \[S20]
* **TSR-102:** Shared (isomorphic) code MUST depend only on the intersection of supported APIs, and MUST not access globals (window, process) without guards.

## 15. Data validation and serialization (zod/io-ts patterns, schema validation discipline)

Static types do not validate runtime data. Validation MUST exist where data enters the system.

* **TSR-103:** All boundary inputs MUST be validated at runtime before use (network responses, requests, env vars, localStorage, DB rows, message queues). \[S38]
* **TSR-104:** Validated data MUST be represented as a strongly typed value derived from the schema (no parallel hand-written types that can drift). \[S38]
* **TSR-105:** Parsing MUST return either a typed value or a typed error. Throwing parsers MAY be used only if the caller has a clear error boundary.
* **TSR-106:** Schema validation MUST be the only place where `unknown` becomes a trusted domain type; other modules MUST accept already-validated inputs.
* **TSR-107:** JSON serialization/deserialization MUST be centralized for each wire format and MUST explicitly handle dates, bigint, and custom classes (no implicit `JSON.stringify` on rich objects).

## 16. Package.json and dependency hygiene (exports, types, peer deps, versioning)

Packaging is part of the API. Dependencies are part of the attack surface.

* **TSR-108:** Every package MUST have a correct `package.json` and MUST declare entrypoints via `exports` for any package intended for consumption. \[S20]\[S29]
* **TSR-109:** Packages MUST provide TypeScript typings via `types` (or via `exports` conditions that point to `.d.ts`) and MUST ensure emitted `.d.ts` paths match runtime JS paths. \[S29]\[S12]
* **TSR-110:** Packages MUST NOT rely on deep imports into their dependencies; only documented entrypoints are allowed (respects dependency encapsulation via `exports`). \[S20]
* **TSR-111:** Packages MUST distinguish dependency types: `dependencies` for runtime, `devDependencies` for build/test, `peerDependencies` for host-provided frameworks/plugins.
* **TSR-112:** Published packages MUST define `files` and/or use `exports` so that build artifacts are published and sources/secrets are not accidentally published. \[S29]\[S20]
* **TSR-113:** Versioning MUST follow SemVer; do not publish modified contents under the same version. \[S39]
* **TSR-114:** If a package supports multiple module formats or runtimes, it MUST express that via conditional exports and MUST include tests that execute each condition. \[S20]

## 17. Testing (unit, integration, deterministic testing, mocking boundaries)

Tests are part of governance: they prevent regressions and document behavior.

* **TSR-115:** Repos MUST run tests in CI and MUST fail on test failures.
* **TSR-116:** Tests MUST be deterministic: they MUST NOT depend on real time, randomness, network, or shared global state without explicit control.
* **TSR-117:** External effects (network, filesystem, timers) MUST be abstracted behind adapters so tests can run without real side effects.
* **TSR-118:** Each package SHOULD have unit tests for pure logic and integration tests for boundary code (I/O, DB, HTTP).
* **TSR-119:** If using Node’s built-in test runner, tests MUST follow its supported patterns and be runnable via `node --test`. \[S34]
* **TSR-120:** Mocking MUST be constrained to module boundaries; do not mock internal functions of the unit under test.

## 18. Build systems and bundling (tsc vs swc vs esbuild vs bundlers as rules)

Build output MUST be reproducible and MUST not hide type errors.

* **TSR-121:** Type checking MUST always be performed by `tsc` (or by an equivalent TypeScript typechecker) as a dedicated step, even if emitting is done by a bundler. \[S32]\[S33]
* **TSR-122:** If using esbuild for transpile/bundle, the repo MUST still run `tsc --noEmit` because esbuild ignores TypeScript types. \[S32]
* **TSR-123:** If using swc for transpile, the repo MUST still run `tsc --noEmit` because swc does not type-check. \[S33]
* **TSR-124:** Libraries that publish types MUST emit `.d.ts` via `tsc` (or a tool that is demonstrably equivalent) and MUST verify that the published `types` resolve for consumers. \[S29]
* **TSR-125:** Project-reference repos MUST build via `tsc --build` and SHOULD use incremental builds for speed. \[S13]\[S15]
* **TSR-126:** The repo MUST provide runnable example commands for typecheck, lint, test, and build. Example:
  \\

```sh
# Typecheck (no emit)
npx tsc --noEmit
# Lint
npx eslint .
# Test (Node built-in runner example)
node --test
# Build (project references)
npx tsc --build
```

\[S13]\[S34]

## 19. CI and quality gates (format, lint, typecheck, test, coverage expectations)

CI is the enforcement layer. If CI is lax, the rules are fiction.

* **TSR-127:** CI MUST run, at minimum, in this order: install (clean), format check, lint, typecheck, tests, build (if producing artifacts). \[S31]
* **TSR-128:** CI MUST use lockfile-driven installs (`npm ci` or equivalent) and MUST not run with floating dependency resolution. \[S31]
* **TSR-129:** CI MUST fail on any TypeScript type error (no `noCheck` builds and no ignoring `tsc` exit codes). \[S48]
* **TSR-130:** Lint and typecheck MUST run against the same set of source files that ship to production.
* **TSR-131:** If coverage is tracked, the repo SHOULD enforce a minimum coverage threshold and MUST prevent decreases without justification.
* **TSR-132:** Release builds MUST be reproducible from a clean checkout and MUST not depend on developer-local state.

## 20. Security and supply-chain rules (dependency pinning, audit, secrets)

Supply-chain attacks are common. Treat dependencies as untrusted code until proven otherwise.

* **TSR-133:** Repos MUST commit a lockfile and MUST use lockfile-based installs in CI. \[S31]
* **TSR-134:** CI MUST run dependency vulnerability scanning (e.g., `npm audit`) at least daily or on every merge to main; high/critical findings MUST block release unless explicitly risk-accepted. \[S30]
* **TSR-135:** New dependencies MUST be justified in the PR description (why needed, why this package, maintenance signals, license) and MUST be minimized.
* **TSR-136:** Packages MUST NOT add `postinstall` scripts or other install-time code execution unless absolutely required and security-reviewed.
* **TSR-137:** Secrets MUST NOT be committed. Repos MUST use secret scanning in CI and MUST rotate any exposed credentials immediately.
* **TSR-138:** When handling untrusted input, code MUST validate and MUST avoid dynamic code execution (`eval`, `new Function`) entirely.

## 21. Anti-patterns list (explicit “do not do this” items)

These are banned patterns because they create runtime surprises, fragile typing, or maintenance debt.

* **TSR-139:** MUST NOT use `// @ts-ignore` or `// @ts-nocheck` except in generated code or as a temporary, ticketed workaround with an expiration date. \[S48]
* **TSR-140:** MUST NOT commit code that type-checks only because of `as any`, `as unknown as T`, or broad `eslint-disable` comments.
* **TSR-141:** MUST NOT disable `strict` in any handwritten code tsconfig. \[S1]
* **TSR-142:** MUST NOT enable `noCheck` in CI typechecking. \[S48]\[S18]
* **TSR-143:** MUST NOT publish packages without an `exports` map when targeting modern Node; do not allow accidental deep imports. \[S20]
* **TSR-144:** MUST NOT mix ESM and CJS semantics in the same file. Use `.mts`/`.cts` (or `.mjs`/`.cjs`) and correct `package.json#type`. \[S19]\[S9]
* **TSR-145:** MUST NOT use `require()` in ESM modules except via a clearly isolated compatibility layer.
* **TSR-146:** MUST NOT use `Array.prototype.forEach(async ...)` (it does not await).
* **TSR-147:** MUST NOT depend on type-only imports being elided unless `verbatimModuleSyntax` is enabled and `import type` is used. \[S7]
* **TSR-148:** MUST NOT set `skipLibCheck: true` in published libraries. \[S47]


## Additional Merged Rules

- **TSR-001:** Code changes MUST keep the repository type-checking clean under the repository’s enforced TypeScript version (5.x+) and tsconfig(s) in CI.
- **TSR-002:** New TypeScript code MUST be written to pass with `strict: true` and the additional strictness flags mandated in this document (see tsconfig discipline). [S1]
- **TSR-003:** All externally obtained data (network, disk, env vars, user input, IPC, database) MUST be treated as `unknown` at the boundary and validated or parsed before use. [S38]
- **TSR-004:** Each distinct runtime environment (Node, DOM, WebWorker, edge) MUST have a dedicated tsconfig with an appropriate `lib` and module settings, connected via project references when in one repo. [S12]
- **TSR-005:** AI agents MUST NOT introduce new `any` types except where explicitly permitted by this ruleset (see Type system rules).
- **TSR-006:** Generated code MUST be isolated (separate folder and tsconfig or excluded) and MUST NOT block CI quality gates for handwritten code.
- **TSR-007:** Repositories MUST pin TypeScript as a devDependency and MUST NOT rely on a globally installed `tsc`.
- **TSR-008:** Repositories MUST specify supported Node.js versions via `package.json#engines.node` and MUST align CI to those versions. Default baseline SHOULD be Node 24 LTS or newer. [S21]
- **TSR-009:** Repositories MUST use a single, workspace-capable package manager across the repo (npm, pnpm, or Yarn) and MUST commit exactly one lockfile (e.g., `package-lock.json`, `pnpm-lock.yaml`, or `yarn.lock`).
- **TSR-010:** CI installs MUST use the package manager’s clean, lockfile-driven install mode (e.g., `npm ci`) and MUST fail if the lockfile is out of sync. [S31]
- **TSR-011:** Repos MUST record the chosen package manager and version using `package.json#packageManager` (Corepack) or equivalent, and CI MUST enforce it.
- **TSR-012:** For TypeScript language changes, repos SHOULD review TypeScript 5.x release notes when upgrading minors, and SHOULD track recent stable 5.x minors unless blocked by a documented compatibility constraint. [S16][S17]
- **TSR-013:** Each publishable package MUST be independently buildable and testable, with its own `package.json`, `tsconfig.json`, and `src/` directory.
- **TSR-014:** Each package MUST treat its `exports` map as the public API surface. Importing another internal file path (deep import) across package boundaries MUST NOT be done. [S20]
- **TSR-015:** Internal-only modules MUST reside under an explicit internal namespace (e.g., `src/internal/**`) and MUST NOT be exported from the package’s `exports` map. [S20]
- **TSR-016:** Cross-layer imports MUST be one-directional (e.g., `app` -> `domain` -> `data`), and cycles across packages or layers MUST NOT be introduced.
- **TSR-017:** Each package MUST have exactly one source root (`rootDir`) and one output root (`outDir`) and MUST NOT emit build artifacts into `src/`.
- **TSR-018:** Monorepos SHOULD use TypeScript project references between packages to enforce layering and to accelerate builds. [S13]
- **TSR-019:** Workspace root MUST contain a build orchestrator entry (scripts or task runner) that can run: format check, lint, typecheck, tests, and build across all packages.
- **TSR-020:** Each package MUST have a `tsconfig.json` checked into version control (no implicit defaults). [S1]
- **TSR-021:** `strict` MUST be enabled in all non-generated code tsconfigs. [S1]
- **TSR-022:** The following strictness flags MUST be enabled in all non-generated code: `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `useUnknownInCatchVariables`, `noPropertyAccessFromIndexSignature`, `noImplicitOverride`. [S2][S3][S4][S6][S5]
- **TSR-023:** `verbatimModuleSyntax: true` MUST be enabled for ESM projects to prevent type/value import confusion and to force `import type` correctness. [S7]
- **TSR-024:** If any code is transpiled by a per-file tool (esbuild, swc, babel), `isolatedModules: true` MUST be enabled in the checked tsconfig used by editors and CI. [S8]
- **TSR-025:** Each referenced project in a project-reference graph MUST set `composite: true`. [S13][S14]
- **TSR-026:** Repos using project references MUST build via `tsc --build` (or equivalent orchestration that invokes it) and MUST NOT rely on ad-hoc `tsc` runs that ignore references. [S13]
- **TSR-027:** All packages in a project-reference graph SHOULD enable `incremental: true` (or `tsc --build` default behavior) and commit no `.tsbuildinfo` files. [S15]
- **TSR-028:** `skipLibCheck` MUST be `false` for libraries intended for reuse; it MAY be set to `true` for applications only with a documented performance rationale and a plan to periodically run a full `skipLibCheck: false` check in CI. [S47]
- **TSR-029:** The repo MUST maintain separate tsconfigs for incompatible runtime libs (e.g., Node-only vs DOM-only code) and connect them with references if they share code. [S12]
- **TSR-030:** Node-targeted ESM packages MUST use `module` modes that integrate with Node’s ESM behavior (e.g., `NodeNext` or `node20`) and MUST align with the package.json `type` and file extensions. [S9][S19]
- **TSR-031:** Bundler-targeted packages (where bundler owns resolution) MAY use `moduleResolution: bundler`, but libraries emitting `.d.ts` SHOULD prefer `moduleResolution: NodeNext` to keep declaration imports compatible for consumers. [S12][S11][S10]
- **TSR-032:** Type checking and emitting MUST be separable. CI MUST have a `tsc --noEmit` step even if the build uses a bundler. [S32][S33]
- **TSR-033:** The repository MUST provide a strict baseline tsconfig. Example baseline:
- **TSR-034:** The repo MUST use an auto-formatter (Prettier preferred) and MUST run it in CI in check mode. [S27][S28]
- **TSR-035:** Formatting configuration MUST be repo-local (checked into version control); relying on developer-global editor settings MUST NOT be done. [S27]
- **TSR-036:** Code MUST NOT be merged if it changes formatting without semantic changes (format first, then refactor).
- **TSR-037:** Exports intended for consumption by other modules SHOULD be named exports; default exports MAY be used only when a framework or platform requires them.
- **TSR-038:** Type-only imports MUST use `import type` (enforced via `verbatimModuleSyntax` and ESLint). [S7]
- **TSR-039:** Imports MUST be grouped in this order: (1) built-ins, (2) external packages, (3) internal packages, (4) relative imports; each group separated by a blank line.
- **TSR-040:** File and directory names MUST be consistent across the repo (choose one: kebab-case or camelCase) and MUST match import casing exactly. [S46]
- **TSR-041:** Public types MUST use `PascalCase`; values MUST use `camelCase`; constants MAY use `SCREAMING_SNAKE_CASE` only for true constants (no runtime mutation).
- **TSR-042:** The repo MUST use ESLint for JavaScript/TypeScript linting and MUST run it in CI. [S25]
- **TSR-043:** The repo MUST use typescript-eslint for TypeScript-aware linting (parser + plugin). [S22][S23]
- **TSR-044:** CI linting MUST run with type information enabled for packages that produce runtime code (use a typed config such as `strict-type-checked`). [S22][S23]
- **TSR-045:** ESLint configuration MUST use the supported config system for the repo (Flat Config preferred for new repos) and MUST be committed. [S25]
- **TSR-046:** Typed linting in monorepos SHOULD use `parserOptions.projectService` to align ESLint type information with editor behavior and project references. [S24]
- **TSR-047:** Rules that can produce false positives at scale MUST start at severity `warn` and MUST be promoted to `error` only once fixed repo-wide. [S26]
- **TSR-048:** The lint configuration MUST disable conflicting core rules in favor of their TypeScript-aware equivalents (example: use `@typescript-eslint/no-unused-vars` instead of `no-unused-vars`). [S23]
- **TSR-049:** The repo MUST provide an ESLint config example equivalent to the following (adjust file globs per repo):
- **TSR-050:** ESLint MUST NOT be used as a substitute for TypeScript type-checking; both MUST run (they detect different categories of issues).
- **TSR-051:** New code MUST NOT introduce explicit `any` (including `any[]`, `Record<string, any>`, `as any`, or implicit anys). Use `unknown` and narrow instead. [S1]
- **TSR-052:** If `any` is unavoidable (interop with untyped third-party), it MUST be confined to a single boundary module and MUST be converted to `unknown` immediately after ingestion.
- **TSR-053:** Type assertions (`as T`) MUST be preceded by a runtime check or schema parse that makes the assertion true, or the assertion MUST be replaced by a parser/validator.
- **TSR-054:** Generic type parameters MUST be constrained when possible (e.g., `T extends Foo`) and MUST NOT be left unconstrained when the implementation assumes structure.
- **TSR-055:** Public generics MUST be simple enough to understand from the signature alone; if the type requires multi-screen conditional gymnastics, it MUST be hidden behind a named helper type or refactored.
- **TSR-056:** Code SHOULD prefer `satisfies` for validating object literals against a type without widening the inferred type, and SHOULD avoid `as` for that use case.
- **TSR-057:** Mapped/conditional types in hot paths of the developer experience (build times, editor) SHOULD be minimized; prefer simpler, explicit types for exported APIs.
- **TSR-058:** `strictNullChecks` MUST be enabled (via `strict`). [S1]
- **TSR-059:** `noUncheckedIndexedAccess` MUST be enabled and code MUST handle the resulting `undefined` in index access results via narrowing or defaults. [S2]
- **TSR-060:** `exactOptionalPropertyTypes` MUST be enabled and code MUST NOT treat `prop?: T` as `T | undefined` unless `undefined` is explicitly part of `T`. [S3]
- **TSR-061:** Non-null assertions (`expr!`) MUST NOT be used in production code except in a tiny, audited boundary where a runtime invariant is proven and documented.
- **TSR-062:** Switch statements over discriminated unions MUST be exhaustive. Use a `never`-based `assertNever(x)` helper and ensure the default branch is unreachable.
- **TSR-063:** Code SHOULD narrow unions using `typeof`, `instanceof`, `in`, and user-defined type guards before accessing properties on `unknown` or union types.
- **TSR-064:** Every package MUST declare a public API via `package.json#exports`; only exported entrypoints are considered supported. [S20][S29]
- **TSR-065:** Public entrypoints MUST be minimal. Re-export only what is intended for consumers; avoid accidental exports via `export *` from internal modules.
- **TSR-066:** Public types MUST avoid leaking private implementation details (e.g., concrete classes, internal discriminants) unless part of the contract.
- **TSR-067:** Published packages MUST follow Semantic Versioning, and breaking changes MUST only ship in a major version bump. [S39]
- **TSR-068:** Any planned breaking change MUST first be introduced as a deprecated API (`/** @deprecated */`) with a migration path, then removed in a major version. [S39]
- **TSR-069:** If the repo uses TypeScript’s deprecation warnings, it MAY use `ignoreDeprecations` only as a temporary upgrade aid and MUST not silence deprecations indefinitely. [S49]
- **TSR-070:** Code MUST throw only `Error` (or subclasses). Throwing strings, numbers, or plain objects MUST NOT be done.
- **TSR-071:** Library code MUST NOT swallow errors; it MUST either (a) rethrow, (b) wrap with causal context, or (c) return an explicit `Result` type when the error is an expected business outcome.
- **TSR-072:** Application code MAY use exceptions for control flow only at process boundaries (e.g., request handler), and MUST convert them to appropriate responses/logs.
- **TSR-073:** Catch variables MUST be treated as `unknown` and narrowed before use (enforce via `useUnknownInCatchVariables`). [S4]
- **TSR-074:** Error messages MUST include actionable context (operation, key identifiers) and MUST NOT leak secrets (tokens, credentials, PII).
- **TSR-075:** When using `never` for exhaustiveness, the code MUST ensure the `never` path is unreachable at runtime (use an `assertNever` that throws).
- **TSR-076:** Async entrypoints (HTTP handlers, jobs, event handlers) MUST have a top-level error boundary that logs and maps errors to a stable output.
- **TSR-077:** Async functions MUST return `Promise<T>` (implicitly via `async`) and MUST NOT mix callback-style APIs without an explicit wrapper.
- **TSR-078:** New async code SHOULD use `async`/`await` for readability; chaining `.then()` MAY be used only for simple transformations where errors are still correctly propagated.
- **TSR-079:** Fire-and-forget Promises MUST be explicitly marked and handled: either `await` them, or use `void promise.catch(handle)`; leaving floating Promises MUST NOT be done (enforce via ESLint). [S23]
- **TSR-080:** `Array.prototype.forEach` MUST NOT be used with `async` callbacks. Use `for...of` (sequential) or `Promise.all`/`Promise.allSettled` (concurrent).
- **TSR-081:** Concurrency MUST be explicit: use `Promise.all()` when tasks are independent and failure should fail-fast. [S36]
- **TSR-082:** Code SHOULD use `Promise.allSettled()` when it needs results from all tasks regardless of individual failures, and MUST handle each outcome explicitly. [S37]
- **TSR-083:** APIs that can be long-running or user-cancelable MUST accept an `AbortSignal` and MUST respect cancellation promptly. [S35]
- **TSR-084:** New cancellation-capable code MUST use `AbortController`/`AbortSignal` (not custom boolean flags) unless a platform forbids it. [S35]
- **TSR-085:** Any performance-motivated change MUST include a benchmark or measurement methodology in the PR description (before/after).
- **TSR-086:** Code in hot paths MUST avoid unnecessary allocations (e.g., avoid creating arrays/objects inside tight loops) unless measured and acceptable.
- **TSR-087:** Do not rely on V8 engine quirks. Performance-sensitive code MUST target correctness first, then measured optimization.
- **TSR-088:** Performance-sensitive code MUST NOT rely on JavaScript engine quirks; it MUST be correct first, then optimized based on measurement.
- **TSR-089:** Exported types SHOULD NOT use unions with hundreds of members; prefer data-driven runtime tables and simple types.
- **TSR-090:** New Node packages SHOULD be ESM-first: set `package.json#type` to `"module"` and emit ESM, unless a documented consumer constraint requires CommonJS. [S19][S29]
- **TSR-091:** Node packages MUST use Node’s explicit markers for module format: `.mjs`/`.cjs` extensions or `package.json#type`. Mixed, implicit behavior MUST NOT be relied on. [S19]
- **TSR-092:** TypeScript `module` and `moduleResolution` MUST be set to a Node-integrated mode (`NodeNext` or `node20`) when targeting Node’s ESM/CJS behavior. [S9]
- **TSR-093:** Relative imports in Node ESM output MUST be resolvable by Node at runtime. When emitting ESM, relative import specifiers SHOULD include file extensions as required by Node’s ESM resolver. [S19][S9]
- **TSR-094:** Packages intended to support both `import` and `require` MUST use conditional exports (`exports` map) and MUST test both entrypoints in CI. [S20]
- **TSR-095:** Interop with CommonJS MUST follow TypeScript’s documented ESM/CJS interoperability guidance; do not assume default import behavior without verifying runtime semantics. [S40]
- **TSR-096:** The `exports` map MUST NOT expose deep internal paths; only explicit entrypoints are allowed. [S20]
- **TSR-097:** Internal path aliases for in-package imports MAY use the `imports` field with `#`-prefixed specifiers; external-facing aliases MUST NOT use `imports`. [S20]
- **TSR-098:** Each runtime target MUST have its own tsconfig with correct `lib` and (when relevant) `types`, and those tsconfigs MUST be used by both editor tooling and CI. [S12]
- **TSR-099:** Browser-targeted code MUST NOT import Node-only built-ins (fs, path, crypto Node APIs) unless behind a runtime-guarded, conditionally exported module.
- **TSR-100:** Edge/serverless code MUST avoid Node APIs not supported by the target platform; platform-specific shims MUST be isolated behind adapter interfaces.
- **TSR-101:** Multi-runtime libraries MUST provide separate entrypoints per runtime using conditional exports (e.g., `browser`, `node`) and MUST test each entrypoint. [S20]
- **TSR-102:** Shared (isomorphic) code MUST depend only on the intersection of supported APIs, and MUST not access globals (window, process) without guards.
- **TSR-103:** All boundary inputs MUST be validated at runtime before use (network responses, requests, env vars, localStorage, DB rows, message queues). [S38]
- **TSR-104:** Validated data MUST be represented as a strongly typed value derived from the schema (no parallel hand-written types that can drift). [S38]
- **TSR-105:** Parsing MUST return either a typed value or a typed error. Throwing parsers MAY be used only if the caller has a clear error boundary.
- **TSR-106:** Schema validation MUST be the only place where `unknown` becomes a trusted domain type; other modules MUST accept already-validated inputs.
- **TSR-107:** JSON serialization/deserialization MUST be centralized for each wire format and MUST explicitly handle dates, bigint, and custom classes (no implicit `JSON.stringify` on rich objects).
- **TSR-108:** Every package MUST have a correct `package.json` and MUST declare entrypoints via `exports` for any package intended for consumption. [S20][S29]
- **TSR-109:** Packages MUST provide TypeScript typings via `types` (or via `exports` conditions that point to `.d.ts`) and MUST ensure emitted `.d.ts` paths match runtime JS paths. [S29][S12]
- **TSR-110:** Packages MUST NOT rely on deep imports into their dependencies; only documented entrypoints are allowed (respects dependency encapsulation via `exports`). [S20]
- **TSR-111:** Packages MUST distinguish dependency types: `dependencies` for runtime, `devDependencies` for build/test, `peerDependencies` for host-provided frameworks/plugins.
- **TSR-112:** Published packages MUST define `files` and/or use `exports` so that build artifacts are published and sources/secrets are not accidentally published. [S29][S20]
- **TSR-113:** Versioning MUST follow SemVer; do not publish modified contents under the same version. [S39]
- **TSR-114:** If a package supports multiple module formats or runtimes, it MUST express that via conditional exports and MUST include tests that execute each condition. [S20]
- **TSR-115:** Repos MUST run tests in CI and MUST fail on test failures.
- **TSR-116:** Tests MUST be deterministic: they MUST NOT depend on real time, randomness, network, or shared global state without explicit control.
- **TSR-117:** External effects (network, filesystem, timers) MUST be abstracted behind adapters so tests can run without real side effects.
- **TSR-118:** Each package SHOULD have unit tests for pure logic and integration tests for boundary code (I/O, DB, HTTP).
- **TSR-119:** If using Node’s built-in test runner, tests MUST follow its supported patterns and be runnable via `node --test`. [S34]
- **TSR-120:** Mocking MUST be constrained to module boundaries; do not mock internal functions of the unit under test.
- **TSR-121:** Type checking MUST always be performed by `tsc` (or by an equivalent TypeScript typechecker) as a dedicated step, even if emitting is done by a bundler. [S32][S33]
- **TSR-122:** If using esbuild for transpile/bundle, the repo MUST still run `tsc --noEmit` because esbuild ignores TypeScript types. [S32]
- **TSR-123:** If using swc for transpile, the repo MUST still run `tsc --noEmit` because swc does not type-check. [S33]
- **TSR-124:** Libraries that publish types MUST emit `.d.ts` via `tsc` (or a tool that is demonstrably equivalent) and MUST verify that the published `types` resolve for consumers. [S29]
- **TSR-125:** Project-reference repos MUST build via `tsc --build` and SHOULD use incremental builds for speed. [S13][S15]
- **TSR-126:** The repo MUST provide runnable example commands for typecheck, lint, test, and build. Example:
- **TSR-127:** CI MUST run, at minimum, in this order: install (clean), format check, lint, typecheck, tests, build (if producing artifacts). [S31]
- **TSR-128:** CI MUST use lockfile-driven installs (`npm ci` or equivalent) and MUST not run with floating dependency resolution. [S31]
- **TSR-129:** CI MUST fail on any TypeScript type error (no `noCheck` builds and no ignoring `tsc` exit codes). [S48]
- **TSR-130:** Lint and typecheck MUST run against the same set of source files that ship to production.
- **TSR-131:** If coverage is tracked, the repo SHOULD enforce a minimum coverage threshold and MUST prevent decreases without justification.
- **TSR-132:** Release builds MUST be reproducible from a clean checkout and MUST not depend on developer-local state.
- **TSR-133:** Repos MUST commit a lockfile and MUST use lockfile-based installs in CI. [S31]
- **TSR-134:** CI MUST run dependency vulnerability scanning (e.g., `npm audit`) at least daily or on every merge to main; high/critical findings MUST block release unless explicitly risk-accepted. [S30]
- **TSR-135:** New dependencies MUST be justified in the PR description (why needed, why this package, maintenance signals, license) and MUST be minimized.
- **TSR-136:** Packages MUST NOT add `postinstall` scripts or other install-time code execution unless absolutely required and security-reviewed.
- **TSR-137:** Secrets MUST NOT be committed. Repos MUST use secret scanning in CI and MUST rotate any exposed credentials immediately.
- **TSR-138:** When handling untrusted input, code MUST validate and MUST avoid dynamic code execution (`eval`, `new Function`) entirely.
- **TSR-139:** MUST NOT use `// @ts-ignore` or `// @ts-nocheck` except in generated code or as a temporary, ticketed workaround with an expiration date. [S48]
- **TSR-140:** MUST NOT commit code that type-checks only because of `as any`, `as unknown as T`, or broad `eslint-disable` comments.
- **TSR-141:** MUST NOT disable `strict` in any handwritten code tsconfig. [S1]
- **TSR-142:** MUST NOT enable `noCheck` in CI typechecking. [S48][S18]
- **TSR-143:** MUST NOT publish packages without an `exports` map when targeting modern Node; do not allow accidental deep imports. [S20]
- **TSR-144:** MUST NOT mix ESM and CJS semantics in the same file. Use `.mts`/`.cts` (or `.mjs`/`.cjs`) and correct `package.json#type`. [S19][S9]
- **TSR-145:** MUST NOT use `require()` in ESM modules except via a clearly isolated compatibility layer.
- **TSR-146:** MUST NOT use `Array.prototype.forEach(async ...)` (it does not await).
- **TSR-147:** MUST NOT depend on type-only imports being elided unless `verbatimModuleSyntax` is enabled and `import type` is used. [S7]
- **TSR-148:** MUST NOT set `skipLibCheck: true` in published libraries. [S47]
- TSR-001: Code changes MUST keep the project type-checking clean under the project's enforced TypeScript version (5.x+) and tsconfig(s) in CI.
- TSR-002: New TypeScript code MUST be written to pass with `strict: true` and the additional strictness flags mandated in this document (see tsconfig discipline). [S1]
- TSR-003: All externally obtained data (network, disk, env vars, user input, IPC, database) MUST be treated as `unknown` at the boundary and validated or parsed before use. [S38]
- TSR-004: Each distinct runtime environment (Node, DOM, WebWorker, edge) MUST have a dedicated tsconfig with an appropriate `lib` and module settings, connected via project references when in one project. [S12]
- TSR-005: AI agents MUST NOT introduce new `any` types except where explicitly permitted by this ruleset (see Type system rules).
- TSR-006: Generated code MUST be isolated (separate folder and tsconfig or excluded) and MUST NOT block CI quality gates for handwritten code.
- TSR-007: Repositories MUST pin TypeScript as a devDependency and MUST NOT rely on a globally installed `tsc`.
- TSR-008: Repositories MUST specify supported Node.js versions via `package.json#engines.node` and MUST align CI to those versions. Default baseline SHOULD be Node 24 LTS or newer. [S21]
- TSR-009: Repositories MUST use a single, workspace-capable package manager across the project (npm, pnpm, or Yarn) and MUST commit exactly one lockfile (e.g., `package-lock.json`, `pnpm-lock.yaml`, or `yarn.lock`).
- TSR-010: CI installs MUST use the package manager’s clean, lockfile-driven install mode (e.g., `npm ci`) and MUST fail if the lockfile is out of sync. [S31]
- TSR-011: Projects MUST record the chosen package manager and version using `package.json#packageManager` (Corepack) or equivalent, and CI MUST enforce it.
- TSR-012: For TypeScript language changes, projects SHOULD review TypeScript 5.x release notes when upgrading minors, and SHOULD track recent stable 5.x minors unless blocked by a documented compatibility constraint. [S16][S17]
- TSR-013: Each publishable package MUST be independently buildable and testable, with its own `package.json`, `tsconfig.json`, and `src/` directory.
- TSR-014: Each package MUST treat its `exports` map as the public API surface. Importing another internal file path (deep import) across package boundaries MUST NOT be done. [S20]
- TSR-015: Internal-only modules MUST reside under an explicit internal namespace (e.g., `src/internal/**`) and MUST NOT be exported from the package’s `exports` map. [S20]
- TSR-016: Cross-layer imports MUST be one-directional (e.g., `app` -> `domain` -> `data`), and cycles across packages or layers MUST NOT be introduced.
- TSR-017: Each package MUST have exactly one source root (`rootDir`) and one output root (`outDir`) and MUST NOT emit build artifacts into `src/`.
- TSR-018: Monorepos SHOULD use TypeScript project references between packages to enforce layering and to accelerate builds. [S13]
- TSR-019: Workspace root MUST contain a build orchestrator entry (scripts or task runner) that can run: format check, lint, typecheck, tests, and build across all packages.
- TSR-020: Each package MUST have a `tsconfig.json` checked into version control (no implicit defaults). [S1]
- TSR-021: `strict` MUST be enabled in all non-generated code tsconfigs. [S1]
- TSR-022: The following strictness flags MUST be enabled in all non-generated code: `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `useUnknownInCatchVariables`, `noPropertyAccessFromIndexSignature`, `noImplicitOverride`. [S2][S3][S4][S6][S5]
- TSR-023: `verbatimModuleSyntax: true` MUST be enabled for ESM projects to prevent type/value import confusion and to force `import type` correctness. [S7]
- TSR-024: If any code is transpiled by a per-file tool (esbuild, swc, babel), `isolatedModules: true` MUST be enabled in the checked tsconfig used by editors and CI. [S8]
- TSR-025: Each referenced project in a project-reference graph MUST set `composite: true`. [S13][S14]
- TSR-026: Projects using project references MUST build via `tsc --build` (or equivalent orchestration that invokes it) and MUST NOT rely on ad-hoc `tsc` runs that ignore references. [S13]
- TSR-027: All packages in a project-reference graph SHOULD enable `incremental: true` (or `tsc --build` default behavior) and commit no `.tsbuildinfo` files. [S15]
- TSR-028: `skipLibCheck` MUST be `false` for libraries intended for reuse; it MAY be set to `true` for applications only with a documented performance rationale and a plan to periodically run a full `skipLibCheck: false` check in CI. [S47]
- TSR-029: The project MUST maintain separate tsconfigs for incompatible runtime libs (e.g., Node-only vs DOM-only code) and connect them with references if they share code. [S12]
- TSR-030: Node-targeted ESM packages MUST use `module` modes that integrate with Node’s ESM behavior (e.g., `NodeNext` or `node20`) and MUST align with the package.json `type` and file extensions. [S9][S19]
- TSR-031: Bundler-targeted packages (where bundler owns resolution) MAY use `moduleResolution: bundler`, but libraries emitting `.d.ts` SHOULD prefer `moduleResolution: NodeNext` to keep declaration imports compatible for consumers. [S12][S11][S10]
- TSR-032: Type checking and emitting MUST be separable. CI MUST have a `tsc --noEmit` step even if the build uses a bundler. [S32][S33]
- TSR-033: The project MUST provide a strict baseline tsconfig. Example baseline:
- TSR-034: The project MUST use an auto-formatter (Prettier preferred) and MUST run it in CI in check mode. [S27][S28]
- TSR-035: Formatting configuration MUST be project-local (checked into version control); relying on developer-global editor settings MUST NOT be done. [S27]
- TSR-036: Code MUST NOT be merged if it changes formatting without semantic changes (format first, then refactor).
- TSR-037: Exports intended for consumption by other modules SHOULD be named exports; default exports MAY be used only when a framework or platform requires them.
- TSR-038: Type-only imports MUST use `import type` (enforced via `verbatimModuleSyntax` and ESLint). [S7]
- TSR-039: Imports MUST be grouped in this order: (1) built-ins, (2) external packages, (3) internal packages, (4) relative imports; each group separated by a blank line.
- TSR-040: File and directory names MUST be consistent across the project (choose one: kebab-case or camelCase) and MUST match import casing exactly. [S46]
- TSR-041: Public types MUST use `PascalCase`; values MUST use `camelCase`; constants MAY use `SCREAMING_SNAKE_CASE` only for true constants (no runtime mutation).
- TSR-042: The project MUST use ESLint for JavaScript/TypeScript linting and MUST run it in CI. [S25]
- TSR-043: The project MUST use typescript-eslint for TypeScript-aware linting (parser + plugin). [S22][S23]
- TSR-044: CI linting MUST run with type information enabled for packages that produce runtime code (use a typed config such as `strict-type-checked`). [S22][S23]
- TSR-045: ESLint configuration MUST use the supported config system for the project (Flat Config preferred for new projects) and MUST be committed. [S25]
- TSR-046: Typed linting in monorepos SHOULD use `parserOptions.projectService` to align ESLint type information with editor behavior and project references. [S24]
- TSR-047: Rules that can produce false positives at scale MUST start at severity `warn` and MUST be promoted to `error` only once fixed project-wide. [S26]
- TSR-048: The lint configuration MUST disable conflicting core rules in favor of their TypeScript-aware equivalents (example: use `@typescript-eslint/no-unused-vars` instead of `no-unused-vars`). [S23]
- TSR-049: The project MUST provide an ESLint config example equivalent to the following (adjust file globs per project):
- TSR-050: ESLint MUST NOT be used as a substitute for TypeScript type-checking; both MUST run (they detect different categories of issues).
- TSR-051: New code MUST NOT introduce explicit `any` (including `any[]`, `Record<string, any>`, `as any`, or implicit anys). Use `unknown` and narrow instead. [S1]
- TSR-052: If `any` is unavoidable (interop with untyped third-party), it MUST be confined to a single boundary module and MUST be converted to `unknown` immediately after ingestion.
- TSR-053: Type assertions (`as T`) MUST be preceded by a runtime check or schema parse that makes the assertion true, or the assertion MUST be replaced by a parser/validator.
- TSR-054: Generic type parameters MUST be constrained when possible (e.g., `T extends Foo`) and MUST NOT be left unconstrained when the implementation assumes structure.
- TSR-055: Public generics MUST be simple enough to understand from the signature alone; if the type requires multi-screen conditional gymnastics, it MUST be hidden behind a named helper type or refactored.
- TSR-056: Code SHOULD prefer `satisfies` for validating object literals against a type without widening the inferred type, and SHOULD avoid `as` for that use case.
- TSR-057: Mapped/conditional types in hot paths of the developer experience (build times, editor) SHOULD be minimized; prefer simpler, explicit types for exported APIs.
- TSR-058: `strictNullChecks` MUST be enabled (via `strict`). [S1]
- TSR-059: `noUncheckedIndexedAccess` MUST be enabled and code MUST handle the resulting `undefined` in index access results via narrowing or defaults. [S2]
- TSR-060: `exactOptionalPropertyTypes` MUST be enabled and code MUST NOT treat `prop?: T` as `T | undefined` unless `undefined` is explicitly part of `T`. [S3]
- TSR-061: Non-null assertions (`expr!`) MUST NOT be used in production code except in a tiny, audited boundary where a runtime invariant is proven and documented.
- TSR-062: Switch statements over discriminated unions MUST be exhaustive. Use a `never`-based `assertNever(x)` helper and ensure the default branch is unreachable.
- TSR-063: Code SHOULD narrow unions using `typeof`, `instanceof`, `in`, and user-defined type guards before accessing properties on `unknown` or union types.
- TSR-064: Every package MUST declare a public API via `package.json#exports`; only exported entrypoints are considered supported. [S20][S29]
- TSR-065: Public entrypoints MUST be minimal. Re-export only what is intended for consumers; avoid accidental exports via `export *` from internal modules.
- TSR-066: Public types MUST avoid leaking private implementation details (e.g., concrete classes, internal discriminants) unless part of the contract.
- TSR-067: Published packages MUST follow Semantic Versioning, and breaking changes MUST only ship in a major version bump. [S39]
- TSR-068: Any planned breaking change MUST first be introduced as a deprecated API (`/** @deprecated */`) with a migration path, then removed in a major version. [S39]
- TSR-069: If the project uses TypeScript’s deprecation warnings, it MAY use `ignoreDeprecations` only as a temporary upgrade aid and MUST not silence deprecations indefinitely. [S49]
- TSR-070: Code MUST throw only `Error` (or subclasses). Throwing strings, numbers, or plain objects MUST NOT be done.
- TSR-071: Library code MUST NOT swallow errors; it MUST either (a) rethrow, (b) wrap with causal context, or (c) return an explicit `Result` type when the error is an expected business outcome.
- TSR-072: Application code MAY use exceptions for control flow only at process boundaries (e.g., request handler), and MUST convert them to appropriate responses/logs.
- TSR-073: Catch variables MUST be treated as `unknown` and narrowed before use (enforce via `useUnknownInCatchVariables`). [S4]
- TSR-074: Error messages MUST include actionable context (operation, key identifiers) and MUST NOT leak secrets (tokens, credentials, PII).
- TSR-075: When using `never` for exhaustiveness, the code MUST ensure the `never` path is unreachable at runtime (use an `assertNever` that throws).
- TSR-076: Async entrypoints (HTTP handlers, jobs, event handlers) MUST have a top-level error boundary
- TSR-076: Async entrypoints (HTTP handlers, jobs, event handlers) MUST have a top-level error boundary that logs and maps errors to a stable output.
- TSR-077: Async functions MUST return `Promise<T>` (implicitly via `async`) and MUST NOT mix callback-style APIs without an explicit wrapper.
- TSR-078: New async code SHOULD use `async`/`await` for readability; chaining `.then()` MAY be used only for simple transformations where errors are still correctly propagated.
- TSR-079: Fire-and-forget Promises MUST be explicitly marked and handled: either `await` them, or use `void promise.catch(handle)`; leaving floating Promises MUST NOT be done (enforce via ESLint). [S23]
- TSR-080: `Array.prototype.forEach` MUST NOT be used with `async` callbacks. Use `for...of` (sequential) or `Promise.all`/`Promise.allSettled` (concurrent).
- TSR-081: Concurrency MUST be explicit: use `Promise.all()` when tasks are independent and failure should fail-fast. [S36]
- TSR-082: Code SHOULD use `Promise.allSettled()` when it needs results from all tasks regardless of individual failures, and MUST handle each outcome explicitly. [S37]
- TSR-083: APIs that can be long-running or user-cancelable MUST accept an `AbortSignal` and MUST respect cancellation promptly. [S35]
- TSR-084: New cancellation-capable code MUST use `AbortController`/`AbortSignal` (not custom boolean flags) unless a platform forbids it. [S35]
- TSR-085: Any performance-motivated change MUST include a benchmark or measurement methodology in the PR description (before/after).
- TSR-086: Code in hot paths MUST avoid unnecessary allocations (e.g., avoid creating arrays/objects inside tight loops) unless measured and acceptable.
- TSR-087: Do not rely on V8 engine quirks. Performance-sensitive code MUST target correctness first, then measured optimization.
- TSR-088: Performance-sensitive code MUST NOT rely on JavaScript engine quirks; it MUST be correct first, then optimized based on measurement.
- TSR-089: Exported types SHOULD NOT use unions with hundreds of members; prefer data-driven runtime tables and simple types.
- TSR-090: New Node packages SHOULD be ESM-first: set `package.json#type` to `"module"` and emit ESM, unless a documented consumer constraint requires CommonJS. [S19][S29]
- TSR-091: Node packages MUST use Node’s explicit markers for module format: `.mjs`/`.cjs` extensions or `package.json#type`. Mixed, implicit behavior MUST NOT be relied on. [S19]
- TSR-092: TypeScript `module` and `moduleResolution` MUST be set to a Node-integrated mode (`NodeNext` or `node20`) when targeting Node’s ESM/CJS behavior. [S9]
- TSR-093: Relative imports in Node ESM output MUST be resolvable by Node at runtime. When emitting ESM, relative import specifiers SHOULD include file extensions as required by Node’s ESM resolver. [S19][S9]
- TSR-094: Packages intended to support both `import` and `require` MUST use conditional exports (`exports` map) and MUST test both entrypoints in CI. [S20]
- TSR-095: Interop with CommonJS MUST follow TypeScript’s documented ESM/CJS interoperability guidance; do not assume default import behavior without verifying runtime semantics. [S40]
- TSR-096: The `exports` map MUST NOT expose deep internal paths; only explicit entrypoints are allowed. [S20]
- TSR-097: Internal path aliases for in-package imports MAY use the `imports` field with `#`-prefixed specifiers; external-facing aliases MUST NOT use `imports`. [S20]
- TSR-098: Each runtime target MUST have its own tsconfig with correct `lib` and (when relevant) `types`, and those tsconfigs MUST be used by both editor tooling and CI. [S12]
- TSR-099: Browser-targeted code MUST NOT import Node-only built-ins (fs, path, crypto Node APIs) unless behind a runtime-guarded, conditionally exported module.
- TSR-100: Edge/serverless code MUST avoid Node APIs not supported by the target platform; platform-specific shims MUST be isolated behind adapter interfaces.
- TSR-101: Multi-runtime libraries MUST provide separate entrypoints per runtime using conditional exports (e.g., `browser`, `node`) and MUST test each entrypoint. [S20]
- TSR-102: Shared (isomorphic) code MUST depend only on the intersection of supported APIs, and MUST not access globals (window, process) without guards.
- TSR-103: All boundary inputs MUST be validated at runtime before use (network responses, requests, env vars, localStorage, DB rows, message queues). [S38]
- TSR-104: Validated data MUST be represented as a strongly typed value derived from the schema (no parallel hand-written types that can drift). [S38]
- TSR-105: Parsing MUST return either a typed value or a typed error. Throwing parsers MAY be used only if the caller has a clear error boundary.
- TSR-106: Schema validation MUST be the only place where `unknown` becomes a trusted domain type; other modules MUST accept already-validated inputs.
- TSR-107: JSON serialization/deserialization MUST be centralized for each wire format and MUST explicitly handle dates, bigint, and custom classes (no implicit `JSON.stringify` on rich objects).
- TSR-108: Every package MUST have a correct `package.json` and MUST declare entrypoints via `exports` for any package intended for consumption. [S20][S29]
- TSR-109: Packages MUST provide TypeScript typings via `types` (or via `exports` conditions that point to `.d.ts`) and MUST ensure emitted `.d.ts` paths match runtime JS paths. [S29][S12]
- TSR-110: Packages MUST NOT rely on deep imports into their dependencies; only documented entrypoints are allowed (respects dependency encapsulation via `exports`). [S20]
- TSR-111: Packages MUST distinguish dependency types: `dependencies` for runtime, `devDependencies` for build/test, `peerDependencies` for host-provided frameworks/plugins.
- TSR-112: Published packages MUST define `files` and/or use `exports` so that build artifacts are published and sources/secrets are not accidentally published. [S29][S20]
- TSR-113: Versioning MUST follow SemVer; do not publish modified contents under the same version. [S39]
- TSR-114: If a package supports multiple module formats or runtimes, it MUST express that via conditional exports and MUST include tests that execute each condition. [S20]
- TSR-115: Projects MUST run tests in CI and MUST fail on test failures.
- TSR-116: Tests MUST be deterministic: they MUST NOT depend on real time, randomness, network, or shared global state without explicit control.
- TSR-117: External effects (network, filesystem, timers) MUST be abstracted behind adapters so tests can run without real side effects.
- TSR-118: Each package SHOULD have unit tests for pure logic and integration tests for boundary code (I/O, DB, HTTP).
- TSR-119: If using Node’s built-in test runner, tests MUST follow its supported patterns and be runnable via `node --test`. [S34]
- TSR-120: Mocking MUST be constrained to module boundaries; do not mock internal functions of the unit under test.
- TSR-121: Type checking MUST always be performed by `tsc` (or by an equivalent TypeScript typechecker) as a dedicated step, even if emitting is done by a bundler. [S32][S33]
- TSR-122: If using esbuild for transpile/bundle, the project MUST still run `tsc --noEmit` because esbuild ignores TypeScript types. [S32]
- TSR-123: If using swc for transpile, the project MUST still run `tsc --noEmit` because swc does not type-check. [S33]
- TSR-124: Libraries that publish types MUST emit `.d.ts` via `tsc` (or a tool that is demonstrably equivalent) and MUST verify that the published `types` resolve for consumers. [S29]
- TSR-125: Project-reference projects MUST build via `tsc --build` and SHOULD use incremental builds for speed. [S13][S15]
- TSR-126: The project MUST provide runnable example commands for typecheck, lint, test, and build. Example:
- TSR-127: CI MUST run, at minimum, in this order: install (clean), format check, lint, typecheck, tests, build (if producing artifacts). [S31]
- TSR-128: CI MUST use lockfile-driven installs (`npm ci` or equivalent) and MUST not run with floating dependency resolution. [S31]
- TSR-129: CI MUST fail on any TypeScript type error (no `noCheck` builds and no ignoring `tsc` exit codes). [S48]
- TSR-130: Lint and typecheck MUST run against the same set of source files that ship to production.
- TSR-131: If coverage is tracked, the project SHOULD enforce a minimum coverage threshold and MUST prevent decreases without justification.
- TSR-132: Release builds MUST be reproducible from a clean checkout and MUST not depend on developer-local state.
- TSR-133: Projects MUST commit a lockfile and MUST use lockfile-based installs in CI. [S31]
- TSR-134: CI MUST run dependency vulnerability scanning (e.g., `npm audit`) at least daily or on every merge to main; high/critical findings MUST block release unless explicitly risk-accepted. [S30]
- TSR-135: New dependencies MUST be justified in the PR description (why needed, why this package, maintenance signals, license) and MUST be minimized.
- TSR-136: Packages MUST NOT add `postinstall` scripts or other install-time code execution unless absolutely required and security-reviewed.
- TSR-137: Secrets MUST NOT be committed. Projects MUST use secret scanning in CI and MUST rotate any exposed credentials immediately.
- TSR-138: When handling untrusted input, code MUST validate and MUST avoid dynamic code execution (`eval`, `new Function`) entirely.
- TSR-139: MUST NOT use `// @ts-ignore` or `// @ts-nocheck` except in generated code or as a temporary, ticketed workaround with an expiration date. [S48]
- TSR-140: MUST NOT commit code that type-checks only because of `as any`, `as unknown as T`, or broad `eslint-disable` comments.
- TSR-141: MUST NOT disable `strict` in any handwritten code tsconfig. [S1]
- TSR-142: MUST NOT enable `noCheck` in CI typechecking. [S48][S18]
- TSR-143: MUST NOT publish packages without an `exports` map when targeting modern Node; do not allow accidental deep imports. [S20]
- TSR-144: MUST NOT mix ESM and CJS semantics in the same file. Use `.mts`/`.cts` (or `.mjs`/`.cjs`) and correct `package.json#type`. [S19][S9]
- TSR-145: MUST NOT use `require()` in ESM modules except via a clearly isolated compatibility layer.
- TSR-146: MUST NOT use `Array.prototype.forEach(async ...)` (it does not await).
- TSR-147: MUST NOT depend on type-only imports being elided unless `verbatimModuleSyntax` is enabled and `import type` is used. [S7]
- TSR-148: MUST NOT set `skipLibCheck: true` in published libraries. [S47]
- **TSR-001:** Code changes MUST keep the project type-checking clean under the project's enforced TypeScript version (5.x+) and tsconfig(s) in CI.
- **TSR-004:** Each distinct runtime environment (Node, DOM, WebWorker, edge) MUST have a dedicated tsconfig with an appropriate `lib` and module settings, connected via project references when in one project. [S12]
- **TSR-009:** Repositories MUST use a single, workspace-capable package manager across the project (npm, pnpm, or Yarn) and MUST commit exactly one lockfile (e.g., `package-lock.json`, `pnpm-lock.yaml`, or `yarn.lock`).
- **TSR-011:** Projects MUST record the chosen package manager and version using `package.json#packageManager` (Corepack) or equivalent, and CI MUST enforce it.
- **TSR-012:** For TypeScript language changes, projects SHOULD review TypeScript 5.x release notes when upgrading minors, and SHOULD track recent stable 5.x minors unless blocked by a documented compatibility constraint. [S16][S17]
- **TSR-026:** Projects using project references MUST build via `tsc --build` (or equivalent orchestration that invokes it) and MUST NOT rely on ad-hoc `tsc` runs that ignore references. [S13]
- **TSR-029:** The project MUST maintain separate tsconfigs for incompatible runtime libs (e.g., Node-only vs DOM-only code) and connect them with references if they share code. [S12]
- **TSR-033:** The project MUST provide a strict baseline tsconfig. Example baseline:
- **TSR-034:** The project MUST use an auto-formatter (Prettier preferred) and MUST run it in CI in check mode. [S27][S28]
- **TSR-035:** Formatting configuration MUST be project-local (checked into version control); relying on developer-global editor settings MUST NOT be done. [S27]
- **TSR-040:** File and directory names MUST be consistent across the project (choose one: kebab-case or camelCase) and MUST match import casing exactly. [S46]
- **TSR-042:** The project MUST use ESLint for JavaScript/TypeScript linting and MUST run it in CI. [S25]
- **TSR-043:** The project MUST use typescript-eslint for TypeScript-aware linting (parser + plugin). [S22][S23]
- **TSR-045:** ESLint configuration MUST use the supported config system for the project (Flat Config preferred for new projects) and MUST be committed. [S25]
- **TSR-047:** Rules that can produce false positives at scale MUST start at severity `warn` and MUST be promoted to `error` only once fixed project-wide. [S26]
- **TSR-049:** The project MUST provide an ESLint config example equivalent to the following (adjust file globs per project):
- **TSR-069:** If the project uses TypeScript’s deprecation warnings, it MAY use `ignoreDeprecations` only as a temporary upgrade aid and MUST not silence deprecations indefinitely. [S49]
- **TSR-115:** Projects MUST run tests in CI and MUST fail on test failures.
- **TSR-122:** If using esbuild for transpile/bundle, the project MUST still run `tsc --noEmit` because esbuild ignores TypeScript types. [S32]
- **TSR-123:** If using swc for transpile, the project MUST still run `tsc --noEmit` because swc does not type-check. [S33]
- **TSR-125:** Project-reference projects MUST build via `tsc --build` and SHOULD use incremental builds for speed. [S13][S15]
- **TSR-126:** The project MUST provide runnable example commands for typecheck, lint, test, and build. Example:
- **TSR-131:** If coverage is tracked, the project SHOULD enforce a minimum coverage threshold and MUST prevent decreases without justification.
- **TSR-133:** Projects MUST commit a lockfile and MUST use lockfile-based installs in CI. [S31]
- **TSR-137:** Secrets MUST NOT be committed. Projects MUST use secret scanning in CI and MUST rotate any exposed credentials immediately.
