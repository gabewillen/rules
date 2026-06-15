---
description: Canonical rules for Hono routing, RPC, and middleware architecture.
trigger: always_on
---

# Hono Rules

- HONO-001: MUST keep path, validation, and handler together in a single chained route definition to preserve type inference. Do not extract standalone controllers.
- HONO-002: MUST organize routes by domain. Mount domain routers as sub-apps using `app.route('/prefix', subApp)`.
- HONO-003: MUST restrict handlers to HTTP transport logic. Extract business logic, database access, and auth policies into dedicated services.
- HONO-004: MUST define child routes before mounting them to prevent silent 404s. Use `app.mount()` ONLY for non-Hono handlers or foreign frameworks.
- HONO-005: MUST define API versioning (e.g., `basePath('/api/v1')`), error shapes, and `notFound()` handling exactly once at the top-level application instance.
- HONO-006: MUST set strict mode (trailing slash policy) once during app creation; do not override per-route.
- HONO-007: MUST treat Hono RPC as typed HTTP. Export exact route-tree types (`export type AppType = typeof routes`) and consume them via `hc<AppType>(baseUrl)`. Do not hand-duplicate types.
- HONO-008: MUST return `c.json(body, literalStatus)` for all RPC outcomes to guarantee client inference. Avoid `c.notFound()` or raw `new Response()` in typed RPC handlers.
- HONO-009: MUST explicitly type bindings and variables on the app instance (`new Hono<Env>()`). Ensure `"strict": true` in all `tsconfig.json` files.
- HONO-010: MUST use `InferRequestType` and `InferResponseType` for `hc` wrappers. Use `$url()` or `$path()` helpers rather than manual path strings.
- HONO-011: MUST use `c.set()` and `c.get()` ONLY for request-scoped data (e.g., auth payloads, request IDs). Never store business state in the Hono `Context`.
- HONO-012: MUST register middleware strictly in this order: 1) Logging/Tracing, 2) Security/CORS/Limits, 3) Auth, 4) Route-level Validation, 5) Handler.
- HONO-013: MUST prefer path-scoped middleware. Do not apply global auth or validation to routes that don't need them.
- HONO-014: MUST leverage built-in middleware (`requestId()`, `secureHeaders()`, `cors()`, `bodyLimit()`) before writing custom implementations.
- HONO-015: MUST use `createMiddleware` or `createFactory<Env>()` to build reusable, strongly-typed middleware stacks.
- HONO-016: MUST use `@hono/zod-validator`. Validate requests (`param`, `query`, `header`, `cookie`, `json`, `form`) before they reach the handler.
- HONO-017: MUST NOT call `await c.req.json()` directly. Read validated input exclusively via `c.req.valid(target)`.
- HONO-018: MUST keep transport schema coercion at the validation boundary. Do not couple transport schemas with domain models.
- HONO-019: MUST require the correct `Content-Type` header when validating JSON or form data. Header validator keys MUST be lowercase.
- HONO-020: MUST standardize on a single JSON error envelope API-wide: `{ error: { code, message, details? }, requestId? }`. Map all internal errors to this shape.
- HONO-021: MUST never expose stack traces or raw exception objects to clients.
- HONO-022: MUST throw `HTTPException` for expected boundary failures inside middleware and transport code.
- HONO-023: MUST use `testClient(app)` for typed RPC tests and `app.request()` for black-box HTTP testing.
- HONO-024: MUST assert status codes, headers, and body shapes for both success and failure cases.
- HONO-025: MUST use `parseResponse()` at the client edge for typed parsing and automatic throwing of non-OK status errors.
- HONO-026: MUST put shared headers in `hc(..., { headers })`. Use per-call headers exclusively for call-specific overrides.
