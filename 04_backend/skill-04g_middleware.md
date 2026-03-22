# SKILL 04-G — Middleware Chains: Order, Responsibility, and Scope
Category: Backend Development
Applies to: Node.js (Express/Fastify/Hono), Python (FastAPI/Django), Go (net/http), NestJS

## What this skill covers
Middleware executes before or after route handlers and provides cross-cutting concerns: authentication, logging, rate limiting, request parsing, CORS, error handling, and caching. Middleware order matters critically — putting authentication after rate limiting or error handling before logging produces incorrect behavior. This skill defines the correct order, the responsibility boundaries for each middleware type, and how to scope middleware globally vs per-route.

## When to activate this skill
- Wiring up middleware for a new backend project
- Debugging a middleware ordering issue (auth, CORS, errors firing in wrong sequence)
- Deciding whether to apply middleware globally or per-route
- Adding a new cross-cutting concern to an existing API

## Core principles
1. **Order determines behavior.** Early middleware sees raw requests; late middleware sees processed ones. A logging middleware that runs after an error handler misses unhandled exceptions.
2. **Global middleware for universal concerns, route-scoped for selective ones.** Rate limiting is global. Role-checking is route-scoped.
3. **Error-handling middleware is always last.** It must catch errors from all preceding middleware and handlers.
4. **Each middleware has one responsibility.** A single middleware that does auth + logging + rate-limiting is untestable and undebuggable.
5. **Request ID must be first.** Every subsequent middleware and handler needs a request ID to correlate log lines.

## Step-by-step guide
**Correct global middleware order (apply in this sequence):**

1. **Request ID** — assign a unique ID to every request (use `crypto.randomUUID()` or `x-request-id` header passthrough)
2. **Structured logger** — log request start with method, path, request ID
3. **CORS** — `Access-Control-Allow-*` headers before any response can short-circuit
4. **Security headers** — Helmet / security header middleware (CSP, HSTS, X-Frame-Options)
5. **Body parser** — parse JSON/form data with size limit
6. **Rate limiter** — applied before auth so unauthenticated requests are also rate-limited
7. **Authentication** — verify token/session, attach user to request context
8. **Route handlers** — actual request processing
9. **Error handler** — catch all errors from handlers, map to HTTP responses, log with request ID

**Route-scoped middleware:**
- Authorization / role checks: applied only to protected route groups
- Input validation: applied per-route or per-router group

## The right way vs the wrong way

| Wrong | Right |
|-------|-------|
| Error handler registered before routes | Error handler must be last — it catches errors thrown by route handlers |
| Auth middleware applied globally before CORS | CORS must come before auth so OPTIONS preflight requests succeed without auth |
| Rate limiter only on `/auth` routes | Apply globally; override with stricter limits on sensitive routes |
| No request ID middleware | First middleware assigns a request ID; all log lines include it |
| Single middleware doing auth + validation + logging | Three separate, testable middleware functions |

## Stack-specific notes
**Express:** Middleware registered with `app.use()` executes in registration order. Error handler signature is `(err, req, res, next)` — 4 args required. Register last.
**Fastify:** `addHook('onRequest', ...)` for early lifecycle. `addHook('preHandler', ...)` for pre-route. `setErrorHandler()` is automatic — no ordering concern.
**NestJS:** Middleware via `configure()` in AppModule, Guards (auth), Interceptors (logging/transform), Pipes (validation), Exception Filters (error). These are distinct extension points with a defined order.
**Go (net/http/chi):** Middleware is a handler wrapper: `func Logging(next http.Handler) http.Handler`. Apply via `r.Use()` in chi — order of calls to `r.Use()` is the execution order.
**FastAPI:** `app.add_middleware()` — last added is outermost. `HTTPSRedirectMiddleware` → `CORSMiddleware` → `GZipMiddleware` → routes.

## Common mistakes
1. **Catching CORS errors in the auth middleware.** Browser sends an OPTIONS preflight — if auth middleware runs first and rejects it, CORS never sets its headers. Always put CORS before auth.
2. **Setting response headers after body is sent.** In Express, middleware that modifies headers must complete before `res.json()` is called by the handler.
3. **Rate limiter only on mutation routes.** Read endpoints can also be used for scraping/data extraction. Apply rate limiting globally.
4. **Forgetting to call `next()` in Express middleware.** Omitting `next()` causes the request to hang silently — no response, no error.

## Checklist
- [ ] Request ID assigned as first middleware
- [ ] Structured logger runs immediately after request ID assignment
- [ ] CORS middleware runs before authentication
- [ ] Security headers (Helmet or equivalent) applied globally
- [ ] Body parser configured with size limit
- [ ] Rate limiter applied globally
- [ ] Auth middleware applies to protected routes after CORS and rate limiting
- [ ] Error handler registered as the last middleware in the chain
- [ ] Each middleware has a single responsibility
- [ ] Route-scoped middleware (role checks, validators) applied only to relevant groups
