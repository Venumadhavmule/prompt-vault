# SKILL 04-F — Serverless and Edge Functions
Category: Backend Development
Applies to: AWS Lambda, Vercel Functions, Cloudflare Workers, Netlify Functions, Node.js, Python, Go

## What this skill covers
Serverless and edge functions eliminate server management but introduce their own constraints: cold starts, execution time limits, stateless request model, limited local filesystem, and different pricing models. This skill covers when to use serverless, how to structure functions, cold start mitigation, state management across invocations, and the differences between Lambda, Vercel Functions, and Cloudflare Workers.

## When to activate this skill
- Building new API routes on Vercel / Netlify / Cloudflare
- Migrating a monolithic backend to serverless functions
- Debugging cold starts or function timeouts
- Choosing between Lambda, Vercel, and Cloudflare Workers

## Core principles
1. **Functions are stateless.** No in-memory state persists between invocations. External stores (Redis, DB, S3) are the only persistent state.
2. **Cold start is a design concern.** Minimize dependencies, use lazy initialization, and co-locate with edge when possible.
3. **Connection pooling must be external.** Direct database connections from Lambda hit the connection limit. Use a connection pooler (RDS Proxy, PgBouncer, Neon serverless driver).
4. **Timeouts are hard limits.** AWS Lambda max: 15 min. Vercel Hobby: 10s, Pro: 60s. Cloudflare Workers: 30s CPU. Design around these or offload to async queues.
5. **Idempotency by default.** Functions can be retried (Lambda retry policies, Vercel retry on timeout). Every mutating operation should be safe to run twice.

## Step-by-step guide
1. **Choose the right platform:**
   - **Cloudflare Workers:** ultra-low latency globally, 0ms cold start, V8 isolates, no Node.js runtime. Best for edge logic, A/B testing, auth middleware.
   - **Vercel Functions:** seamless with Next.js, Node.js runtime, 0–200ms cold start, per-region. Best for API routes in Next.js/Nuxt projects.
   - **AWS Lambda:** most powerful, connects to full AWS ecosystem (SQS, DynamoDB, EventBridge). Best for complex event-driven workloads.
2. **One function, one responsibility.** Avoid a single "catch-all" Lambda that routes all requests internally — it defeats isolation and increases cold start surface.
3. **Minimize bundle size:** tree-shake imports. For Lambda: use ESM + bundler (esbuild), not `node_modules/` zip. For Workers: Wrangler handles bundling.
4. **Manage DB connections with pooling middleware:**
   - Postgres: use `@neondatabase/serverless` (WebSocket driver for Workers) or RDS Proxy for Lambda
   - Redis: use Upstash (REST-based Redis for edge)
5. **Structured response:** always return a proper HTTP status code and content-type header. Missing headers cause browsers and intermediaries to mishandle responses.
6. **Environment variables from platform secrets** — Vercel dashboard, AWS SSM/Secrets Manager, Wrangler secrets. Never in source code.

## The right way vs the wrong way

| Wrong | Right |
|-------|-------|
| Opening a new DB connection on every invocation | Connection pool outside the handler function; reuse across warm invocations |
| Writing to `/tmp` and expecting it to persist | `/tmp` on Lambda is per-instance, not per-invocation. Use S3 for persistent storage |
| Synchronous processing of heavy jobs in a function | Emit to SQS/queue, process async in a background function |
| Cloudflare Worker using Node.js `fs` module | Workers run in V8 isolates — use Web APIs only (fetch, crypto, KV) |
| Not setting timeout on database queries | DB query time eats into function timeout — always set query timeout < function timeout |

## Stack-specific notes
**Node.js (Lambda/Vercel):** Initialize DB outside the handler to reuse across warm invocations: `const db = createDbConnection()` at module level. Use `esbuild` for minimal bundle size.
**Python (Lambda):** Use `psycopg2-binary` layer or `pg8000` for Postgres. Initialize SQLAlchemy engine at module level. Use `mangum` adapter to run FastAPI/Django on Lambda.
**Cloudflare Workers:** Use `import { Hono } from 'hono'` — Hono is Workers-native. Access secrets via `env` param passed to handler, not `process.env`.

## Common mistakes
1. **Putting secrets in function environment variables displayed in console.** For sensitive values, use AWS Secrets Manager or Vercel encrypted secrets — not plaintext env vars in the function config.
2. **Ignoring Lambda concurrency limits.** Default Lambda concurrency is 1000 per region across all functions. Reserve concurrency for critical functions in `aws lambda put-function-concurrency`.
3. **Long-running operations blocking cold start.** SDK initialization at module level should be lazy where possible — eager initialization that takes 2s adds 2s to every cold start.
4. **Missing CORS headers.** Serverless functions must set `Access-Control-Allow-Origin` and handle `OPTIONS` preflight requests, especially when called from a browser frontend.

## Checklist
- [ ] Platform chosen based on runtime requirements (CPU time, latency, ecosystem)
- [ ] DB connections initialized outside the handler (module-level for warm reuse)
- [ ] Connection pooler (RDS Proxy, Neon, Upstash) used for serverless DB access
- [ ] Bundle size minimized via tree-shaking and bundler (esbuild/Wrangler)
- [ ] All secrets loaded from platform secret store (not hardcoded env vars)
- [ ] Function timeout set below max; expensive ops offloaded to async queue
- [ ] Mutating operations are idempotent (safe to retry)
- [ ] CORS headers set correctly for browser-facing functions
- [ ] Structured response with correct HTTP status and content-type
- [ ] Cold start tested in production environment with real traffic profile
