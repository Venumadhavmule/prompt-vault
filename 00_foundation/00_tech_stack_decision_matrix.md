# Tech Stack Decision Matrix

> Use this document to make technology choices based on project requirements.
> Every decision has trade-offs. This matrix makes them explicit.

---

## 1. Frontend Framework

| Criteria                   | Next.js (App Router)      | Remix                      | Vite SPA (React)           |
|----------------------------|---------------------------|----------------------------|----------------------------|
| **Best for**               | Full SaaS products, SEO-critical apps, marketing + app combo | Form-heavy apps, progressive enhancement, nested layouts | Internal tools, dashboards, SPAs behind auth |
| **SSR/SSG**                | Yes (RSC, SSR, SSG, ISR)  | Yes (loader-based SSR)     | No (client-only)           |
| **SEO**                    | Excellent                 | Excellent                  | Poor (requires workarounds)|
| **Bundle size**            | Medium-Large              | Medium                     | Small                      |
| **Learning curve**         | High (RSC model)          | Medium                     | Low                        |
| **Deployment**             | Vercel (easiest), Node, Docker | Node, Docker, Cloudflare | Any static host            |
| **API routes**             | Built-in (/app/api)       | Loaders/Actions            | Separate backend needed    |
| **Real-time**              | Needs separate WS server  | Needs separate WS server   | Needs separate WS server   |
| **When NOT to use**        | Simple internal dashboards| When you need heavy client interactivity | When you need SEO |

**Default choice**: Next.js for most SaaS products.
**Choose Remix**: When you want progressive enhancement and form-centric UX.
**Choose Vite SPA**: When the app is fully behind auth and SEO is irrelevant.

---

## 2. Database

| Criteria                   | PostgreSQL                 | MongoDB                    | SQLite                     |
|----------------------------|----------------------------|----------------------------|----------------------------|
| **Best for**               | Most production apps       | Document-centric, flexible schema | Embedded, edge, prototypes |
| **Data model**             | Relational, strict schema  | Document, flexible schema  | Relational, strict schema  |
| **Transactions**           | Full ACID                  | Multi-doc transactions (4.0+) | Full ACID                |
| **JSON support**           | Excellent (jsonb)          | Native                     | JSON1 extension            |
| **Full-text search**       | Built-in (tsvector)        | Atlas Search               | FTS5 extension             |
| **Scaling**                | Vertical + read replicas   | Horizontal sharding        | Single file, no scaling    |
| **Hosting**                | Neon, Supabase, RDS, self-host | Atlas, self-host        | Local file, Turso (edge)   |
| **When NOT to use**        | Never a wrong choice       | When you need complex joins and strict schemas | Multi-server production |

**Default choice**: PostgreSQL for everything production.
**Choose MongoDB**: When your data is genuinely document-shaped and varies per record.
**Choose SQLite**: For prototypes, CLI tools, edge functions, or embedded apps.

---

## 3. API Style

| Criteria                   | REST                       | tRPC                       | GraphQL                    |
|----------------------------|----------------------------|----------------------------|----------------------------|
| **Best for**               | Public APIs, multi-client  | TypeScript monorepos, internal APIs | Complex data graphs, mobile apps |
| **Type safety**            | Manual (Zod + codegen)     | Automatic end-to-end       | Codegen required           |
| **Learning curve**         | Low                        | Low (if TypeScript)        | High                       |
| **Caching**                | HTTP caching (ETags, CDN)  | React Query integration    | Normalized cache (Apollo)  |
| **File uploads**           | Native multipart           | Needs workaround           | Needs workaround           |
| **Tooling**                | Postman, curl, universal   | TypeScript only            | GraphiQL, Apollo Studio    |
| **Over/under-fetching**    | Common problem             | No (typed procedures)      | Solved by design           |
| **When NOT to use**        | Never a wrong choice       | When clients are not TypeScript | Simple CRUD apps         |

**Default choice**: REST for public APIs and multi-platform clients.
**Choose tRPC**: When frontend and backend are both TypeScript in the same monorepo.
**Choose GraphQL**: When you have deeply nested relational data that multiple clients query differently.

---

## 4. Caching Strategy

| Criteria                   | Redis                      | In-Memory (Map/LRU)       | Database (materialized views) |
|----------------------------|----------------------------|----------------------------|-------------------------------|
| **Best for**               | Sessions, rate limits, queues, shared cache | Single-instance hot data | Expensive query results     |
| **Persistence**            | Optional (RDB/AOF)         | None (lost on restart)     | Full (it is the DB)          |
| **Shared across instances**| Yes                        | No                         | Yes                          |
| **Speed**                  | Sub-millisecond            | Nanosecond                 | Millisecond                  |
| **Data structures**        | Strings, Hashes, Sets, Sorted Sets, Streams | JS objects | SQL tables                |
| **TTL support**            | Native                     | Manual implementation      | Manual refresh               |
| **When NOT to use**        | Single-instance tiny apps  | Multi-server deployments   | Frequently changing data     |

**Default choice**: Redis for all production caching and session storage.
**Choose In-Memory**: For single-server apps or hot data that fits in memory.
**Choose DB**: For expensive analytical queries that change infrequently.

---

## 5. ORM / Query Builder

| Criteria                   | Prisma                     | Drizzle                    | Raw SQL (with pg/mysql2)   |
|----------------------------|----------------------------|----------------------------|----------------------------|
| **Best for**               | Rapid development, schema-first | Performance-critical, SQL-lovers | Complex queries, full control |
| **Type safety**            | Excellent (generated client) | Excellent (schema inference) | Manual (or with pgtyped)  |
| **Migration system**       | Built-in (prisma migrate)  | Built-in (drizzle-kit)     | Manual (node-pg-migrate)   |
| **Query complexity**       | Limited (no raw joins easily) | Full SQL expressiveness   | Unlimited                  |
| **Performance**            | Good (slight overhead)     | Excellent (thin layer)     | Best (no overhead)         |
| **Learning curve**         | Low                        | Medium                     | High                       |
| **Edge runtime**           | Limited                    | Yes (with D1/Turso)        | Depends on driver          |
| **When NOT to use**        | Performance-critical hot paths | When team wants simplicity | When rapid prototyping    |

**Default choice**: Prisma for most projects (fastest to productive).
**Choose Drizzle**: When you need edge compatibility or raw SQL control with type safety.
**Choose Raw SQL**: For complex reporting queries or when ORM is genuinely in the way.

---

## 6. Testing Framework

| Criteria                   | Vitest                     | Jest                       | Playwright                 |
|----------------------------|----------------------------|----------------------------|----------------------------|
| **Best for**               | Unit + integration tests   | Legacy projects already on Jest | E2E browser tests        |
| **Speed**                  | Fast (native ESM, Vite)    | Slower (transform overhead)| N/A (different category)   |
| **TypeScript**             | Native                     | Needs ts-jest or SWC       | Native                     |
| **Watch mode**             | Excellent                  | Good                       | N/A                        |
| **API testing**            | With supertest             | With supertest             | Built-in (request context) |
| **Browser testing**        | No (use Playwright)        | Limited (jsdom)            | Full browser automation    |
| **When NOT to use**        | Legacy CJS-heavy projects  | New projects (use Vitest)  | Unit/integration tests     |

**Default choice**: Vitest for unit + integration. Playwright for E2E.

---

## 7. Backend Framework

| Criteria                   | Hono                       | Fastify                    | Express                    | NestJS                     |
|----------------------------|----------------------------|----------------------------|----------------------------|----------------------------|
| **Best for**               | Edge, serverless, lightweight APIs | High-performance Node APIs | Simple APIs, widespread knowledge | Enterprise, large teams |
| **Performance**            | Excellent                  | Excellent                  | Good                       | Good (Express under hood)  |
| **TypeScript**             | Native, excellent          | Good                       | Needs setup                | Native, excellent          |
| **Runtime support**        | Bun, Deno, CF Workers, Node | Node                      | Node                       | Node                       |
| **Middleware ecosystem**   | Growing                    | Large (via plugins)        | Largest                    | Decorators + modules       |
| **Learning curve**         | Very low                   | Low                        | Very low                   | High (DI, decorators)      |
| **Validation**             | Zod middleware             | JSON Schema (Ajv)          | Manual or express-validator | class-validator + pipes    |
| **When NOT to use**        | Enterprise with strict DI needs | Edge/serverless      | Performance-critical       | Small projects, MVPs       |

**Default choice**: Hono for new APIs (most versatile, fastest, edge-ready).
**Choose Fastify**: When you need mature Node.js plugin ecosystem.
**Choose Express**: When team already knows it and project is simple.
**Choose NestJS**: When building enterprise-grade with large team needing strict architecture.

---

## 8. Package Recommendations by Category

### Validation
- **Zod**: Default choice. TypeScript-first, composable, works everywhere.
- Alternative: Valibot (smaller bundle for client-side).

### Authentication
- **Auth.js (NextAuth v5)**: For Next.js projects with OAuth needs.
- **Custom JWT**: For API-only backends (use jose library for JWT operations).
- **Lucia**: Lightweight, database-agnostic auth library.

### Payments
- **Stripe SDK (@stripe/stripe-node)**: No alternative. Stripe is the standard.
- **LemonSqueezy**: For simpler billing without Stripe complexity (MoR model).

### Email
- **Resend**: Modern email API, excellent DX.
- **React Email**: For building email templates as React components.
- **Nodemailer**: Self-hosted SMTP, most flexible.

### Job Queues
- **BullMQ**: Production-grade Redis-based queue. Default choice.
- **Trigger.dev**: Serverless-friendly background jobs.
- **Inngest**: Event-driven functions with retry, scheduling.

### File Upload
- **UploadThing**: Easiest integration for TypeScript apps.
- **Multer/Busboy**: For self-managed uploads to S3.
- **Sharp**: Image processing (resize, compress, convert).

### Logging
- **Pino**: Fastest structured JSON logger. Default choice.
- **Winston**: More features, slightly slower.

### Rate Limiting
- **@hono/rate-limiter**: For Hono apps.
- **rate-limiter-flexible**: Framework-agnostic, Redis-backed.
- **upstash/ratelimit**: Serverless-friendly, Redis-based.

### State Management (Frontend)
- **Zustand**: Default choice for React. Simple, tiny, scalable.
- **TanStack Query**: For server state (API data caching, revalidation).
- **Jotai**: When you need atomic state management.

### UI Components
- **shadcn/ui**: Default choice. Copy-paste, customizable, Tailwind-based.
- **Radix Primitives**: Unstyled accessible primitives (shadcn is built on this).

### Date/Time
- **date-fns**: Lightweight, tree-shakeable.
- **dayjs**: Moment-compatible API, smaller bundle.

### Monitoring
- **Sentry**: Error tracking + performance monitoring. Default choice.
- **OpenTelemetry**: For distributed tracing (use with Jaeger/Grafana).
- **Prometheus + Grafana**: For metrics dashboards.
