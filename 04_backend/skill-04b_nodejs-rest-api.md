# SKILL 04-B — Production Node.js REST API (Express / Fastify / Hono)
Category: Backend Development
Applies to: Node.js, Express, Fastify, Hono, TypeScript

## What this skill covers
Building a production-ready REST API in Node.js requires more than wiring up routes. This skill covers framework selection, request validation, structured error handling, security headers, rate limiting, graceful shutdown, and the TypeScript configuration needed to ship a reliable service. It covers Express for familiarity, Fastify for performance, and Hono for edge deployment.

## When to activate this skill
- Starting a new Node.js REST API
- Adding production hardening to an existing Express app
- Choosing between Express, Fastify, and Hono
- Debugging slow or crashing Node.js APIs

## Core principles
1. **Validate all input at the boundary.** Use `zod` or Fastify's JSON Schema to validate and type request bodies, query params, and path params before they reach business logic.
2. **Never leak internals in error responses.** Stack traces and internal error messages must never appear in API responses in production.
3. **Async errors must be caught.** Unhandled promise rejections crash Node.js. Always wrap async route handlers or use a framework that handles this natively (Fastify, Hono).
4. **Shutdown is a feature.** The server must drain in-flight requests before exiting to prevent data loss under deployments.
5. **Security headers are baseline, not optional.** Use Helmet (Express) or equivalent to set CSP, HSTS, X-Content-Type-Options, etc. on every response.

## Step-by-step guide
1. **Choose framework by use case:**
   - **Express:** most ecosystem support, requires manual async error wrapping
   - **Fastify:** 2–3× faster than Express, built-in validation via JSON Schema, native async support
   - **Hono:** best for Cloudflare Workers, Bun, Deno edge deployments
2. **Setup TypeScript:** `strict: true`, `moduleResolution: "Bundler"` or `"Node16"`, `target: "ES2022"`.
3. **Parse and validate env at startup** using `zod` (see skill 04-A).
4. **Apply global middleware before routes:**
   - `helmet()` for security headers
   - `compression()` for response compression
   - `express.json({ limit: '1mb' })` for body parsing with size limit
   - Rate limiter (`express-rate-limit` or equivalent)
   - Request logger (pino-http)
5. **Validate request in route handler** using `zod.parse()` (Express) or Fastify schema on the route definition.
6. **Define a global error handler** (Express: 4-argument middleware at the bottom; Fastify: `setErrorHandler`). Map domain errors to HTTP codes. Log stack in dev, never in prod.
7. **Implement graceful shutdown:**
   ```ts
   process.on('SIGTERM', async () => {
     await server.close();
     await db.close();
     process.exit(0);
   });
   ```
8. **Use `pino` for logging** — JSON output, levels, serializers for `req` and `err` objects.

## The right way vs the wrong way

| Wrong | Right |
|-------|-------|
| `app.get('/user', async (req, res) => { ... })` with unhandled async errors in Express | Wrap with `asyncHandler()` utility or use Fastify/Hono which catch async errors natively |
| `res.json({ error: err.message })` in production | Generic message in prod, detailed in dev; always log full error server-side |
| No body size limit | `express.json({ limit: '1mb' })` — prevent payload attacks |
| `return res.status(200)` without `.json()` | Always return a body or explicitly use `res.sendStatus()` |
| Rate limiting only on auth routes | Apply rate limiting globally; stricter limits on auth and expensive routes |

## Stack-specific notes
**Express:** Always wrap async route handlers. Use `express-async-errors` package to patch Express's router, or wrap manually. Helmet v6+ is ESM-only; pin to v5 for CJS projects.
**Fastify:** Define a `schema.body`, `schema.querystring`, `schema.params` per route — Fastify auto-validates and auto-generates faster serialization. Use `fastify-plugin` for shared plugins.
**Hono:** Deploy to Cloudflare Workers with `app.fire()`. Use Hono's built-in `zValidator` middleware for Zod integration. No graceful shutdown needed for serverless/edge.

## Common mistakes
1. **Returning `res.json()` after `res.json()`.** Two response calls cause a "headers already sent" error. Always `return` after sending a response.
2. **Not setting `NODE_ENV=production`.** Express removes error details from responses only when `NODE_ENV` is `production`. Failing to set this leaks stack traces in staging.
3. **`app.listen()` without a callback or error handler.** Port conflicts crash silently. Log on successful bind and handle `EADDRINUSE`.
4. **No request ID.** Without a request ID on every log line, correlating logs across a request chain is impossible. Use `pino-http` which auto-attaches `reqId`.

## Checklist
- [ ] Framework selected with justification (Express / Fastify / Hono)
- [ ] TypeScript strict mode enabled
- [ ] All env vars validated at startup
- [ ] Helmet (or equivalent security headers) applied globally
- [ ] Body parser configured with size limit
- [ ] Rate limiter applied globally
- [ ] All async route errors caught (natively or via wrapper)
- [ ] Global error handler prevents internals from leaking in production
- [ ] Structured JSON logger (pino) with request ID
- [ ] Graceful shutdown on SIGTERM drains requests and closes DB connections
- [ ] All route inputs validated with zod or framework schema
