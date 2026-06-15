---
description: Canonical rules for Hono routing, RPC, and middleware architecture.
trigger: always_on
---

# Hono Rules

## Architecture & Routing
1. **Chained Definitions:** Keep path, validation, and handler together in a single chained route definition to preserve type inference. Do not extract standalone controllers.
2. **Domain-Driven Layout:** Organize routes by domain. Mount domain routers as sub-apps using `app.route('/prefix', subApp)`. 
3. **Thin Handlers:** Restrict handlers to HTTP transport logic. Extract business logic, database access, and auth policies into dedicated services.
4. **Mount Order:** Define child routes before mounting them to prevent silent 404s. Use `app.mount()` ONLY for non-Hono handlers or foreign frameworks.
5. **Centralize Config:** Define API versioning (e.g., `basePath('/api/v1')`), error shapes, and `notFound()` handling exactly once at the top-level application instance.
6. **Trailing Slashes:** Set strict mode (trailing slash policy) once during app creation; do not override per-route.

## Hono RPC & Type Safety
1. **Export Route Trees:** Treat Hono RPC as typed HTTP. Export exact route-tree types (`export type AppType = typeof routes`) and consume them via `hc<AppType>(baseUrl)`. Do not hand-duplicate types.
2. **Typed Responses:** Return `c.json(body, literalStatus)` for all RPC outcomes to guarantee client inference. Avoid `c.notFound()` or raw `new Response()` in typed RPC handlers.
3. **Strict Types:** Explicitly type bindings and variables on the app instance (`new Hono<Env>()`). Ensure `"strict": true` in all `tsconfig.json` files.
4. **RPC Utilities:** Use `InferRequestType` and `InferResponseType` for `hc` wrappers. Use `$url()` or `$path()` helpers rather than manual path strings.

## Middleware & Context
1. **Context Usage:** Use `c.set()` and `c.get()` ONLY for request-scoped data (e.g., auth payloads, request IDs). Never store business state in the Hono `Context`.
2. **Registration Order:** Register middleware strictly: 1) Logging/Tracing, 2) Security/CORS/Limits, 3) Auth, 4) Route-level Validation, 5) Handler.
3. **Scoped Application:** Prefer path-scoped middleware. Do not apply global auth or validation to routes that don't need them.
4. **Built-ins First:** Leverage built-in middleware (`requestId()`, `secureHeaders()`, `cors()`, `bodyLimit()`) before writing custom implementations.
5. **Reusable Factories:** Use `createMiddleware` or `createFactory<Env>()` to build reusable, strongly-typed middleware stacks.

## Validation
1. **Zod Validator:** Use `@hono/zod-validator`. Validate requests (`param`, `query`, `header`, `cookie`, `json`, `form`) before they reach the handler.
2. **No Direct Parsing:** Never call `await c.req.json()` directly. Read validated input exclusively via `c.req.valid(target)`.
3. **Transport Separation:** Keep transport schema coercion at the validation boundary. Do not couple transport schemas with domain models.
4. **Headers & Content-Type:** Require the correct `Content-Type` header when validating JSON or form data. Header validator keys MUST be lowercase.

## Error Handling
1. **Standard Envelope:** Standardize on a single JSON error envelope API-wide: `{ error: { code, message, details? }, requestId? }`. Map all internal errors to this shape.
2. **Never Leak Internals:** Never expose stack traces or raw exception objects to clients.
3. **Expected Exceptions:** Throw `HTTPException` for expected boundary failures inside middleware and transport code.

## Testing & Clients
1. **Test Strategies:** Use `testClient(app)` for typed RPC tests and `app.request()` for black-box HTTP testing.
2. **Exhaustive Assertions:** Assert status codes, headers, and body shapes for both success and failure cases.
3. **Client Parsing:** Use `parseResponse()` at the client edge for typed parsing and automatic throwing of non-OK status errors.
4. **Client Headers:** Put shared headers in `hc(..., { headers })`. Use per-call headers exclusively for call-specific overrides.
