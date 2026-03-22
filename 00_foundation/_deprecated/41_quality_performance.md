# SKILL 4.2 -- How to Optimize Performance Correctly

## What this skill is

This skill teaches the correct process for identifying and fixing performance
problems. The most common performance mistake is optimizing the wrong thing --
adding caching for a CPU-bound problem, or adding indexes for a query that is
slow for a completely different reason. This skill begins with measurement and
ends with documented, verified improvements.

---

## When to use this skill

- When users or monitoring report slow response times
- When a load test reveals under-performance
- Before scaling costs become unacceptable
- When a specific query, route, or operation is flagged as slow
- When planning a new feature that may have performance implications

---

## Full Guide

### The golden rule: measure before you optimize

NEVER optimize by instinct. Optimization without measurement produces:
- Work that does not improve the user experience
- New bugs from unnecessary complexity
- Optimized code that makes no difference to the actual bottleneck

Before changing any code for performance:
1. Capture the baseline metric (response time, query time, CPU usage)
2. Identify the bottleneck (not the obvious place -- the actual measured slow part)
3. Apply one optimization
4. Measure again
5. Compare to baseline
6. Document the improvement

---

### Step 1: Identify the bottleneck category

Performance problems fall into four categories:

| Category | Symptoms | How to identify |
|---|---|---|
| Database | High query count, slow queries, locking | DB query logs, EXPLAIN ANALYZE, query count per request |
| Network | High latency, large payloads | Response time minus DB time, payload size in network tab |
| CPU | High CPU, slow computation | CPU profiler, flamegraph |
| Memory | Increasing memory over time, GC pauses | Memory profiler, heap snapshot |

**For most web applications**: The bottleneck is almost always the database.

Start with database profiling before investigating anything else.

---

### Step 2: Database performance diagnosis

**Step 2a: Find slow queries**

Enable query logging in development:
```typescript
// Prisma
const db = new PrismaClient({
  log: ['query', 'info', 'warn', 'error'],
})

// Log queries taking > 100ms
db.$on('query', (e) => {
  if (e.duration > 100) {
    console.warn(`Slow query (${e.duration}ms):`, e.query)
  }
})
```

In production: use your DB's slow query log:
```sql
-- PostgreSQL: queries taking > 100ms
SELECT query, mean_exec_time, calls
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;
```

**Step 2b: Read EXPLAIN ANALYZE**

For any suspicious query, run EXPLAIN ANALYZE:
```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = '...' ORDER BY created_at DESC LIMIT 20;
```

Reading the output:
- `Seq Scan` on a large table → missing index
- `cost=0.00..8000.00` → high cost, investigate
- `actual time=0.01..450.32` → 450ms actual execution time, very slow
- `Rows Removed by Filter: 95000` → scanning 95k rows to find your results → add index
- `Nested Loop` with large row estimates → potential N+1 or cross join issue

**Step 2c: N+1 queries**

This is the most common database performance bug. It occurs when:
- 1 query fetches N items
- Then N queries each fetch related data for one item

Diagnosis: Enable query counting. If you see "200 queries for one page load" -- you have N+1.

Fix: Use eager loading / `include` in your ORM:
```typescript
// N+1 (200 queries for 100 purchases)
const purchases = await db.purchase.findMany()
for (const p of purchases) {
  p.user = await db.user.findById(p.userId) // 1 query per purchase
}

// Eager load (2 queries total)
const purchases = await db.purchase.findMany({
  include: { user: true }
})
```

---

### Step 3: Index strategy

**When to add an index:**
- Every foreign key column (not auto-indexed in PostgreSQL)
- Every column used in `WHERE` clauses in frequently-run queries
- Every column used in `ORDER BY` for paginated queries
- Every column used in `JOIN` conditions

**When NOT to add an index:**
- Very small tables (< 1000 rows) -- sequential scan is faster
- Columns that are very low cardinality (e.g., boolean `is_active` -- only 2 values)
- Columns that are almost never queried

**Adding indexes safely to live tables:**
```sql
-- BLOCKS all writes (dangerous on large tables):
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- Non-blocking (takes longer but safe for live tables):
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);
```

**Composite index column order:**
Put the most selective filter first:
```sql
-- Query: WHERE org_id = ? AND status = 'active' ORDER BY created_at DESC
-- Correct: org_id first (high cardinality), then status (low cardinality), then created_at for sort
CREATE INDEX ON orders(org_id, status, created_at DESC);
```

---

### Step 4: Caching strategy

**What to cache:**
- Results of expensive queries that change infrequently (user profile, org settings, pricing config)
- Results of external API calls (geocoding, exchange rates, stock prices)
- Aggregated statistics (total user count, revenue totals for dashboards)
- Session data

**What NOT to cache:**
- Data that must be real-time accurate (bank balances, inventory counts)
- Data specific to a single user and fetched infrequently (not worth the eviction complexity)
- Data that changes very frequently (live chat messages, real-time feeds)

**Cache layers and where to use each:**

| Layer | Technology | Speed | Shared? | Use for |
|---|---|---|---|---|
| In-process | Map / LRU cache | Fastest | No (per instance) | Config, hardly-changing data |
| Redis | Redis | Sub-ms | Yes | Sessions, per-user rate counts, shared data |
| CDN | Cloudflare/Fastly | Edge cache | Yes | Static assets, public API responses |
| DB query cache | PostgreSQL query cache | Automatic | Yes | Repeated identical queries |

**Cache invalidation patterns:**
- TTL (time-to-live): simplest, set an expiry, accept slight staleness
- Event-driven: invalidate when the underlying data changes (more complex but always fresh)
- Cache-aside: read from cache, on miss read from DB and populate cache

**Cache stampede prevention:**
When cache expires, many requests all read from DB simultaneously.
Fix: Probabilistic early expiry or mutex lock on cache miss.

---

### Step 5: API performance

**Pagination:** Never return more than 100 items without pagination. Use cursor pagination
for large datasets. Add `LIMIT` to every query.

**Field selection:** Allow clients to request only the fields they need:
```
GET /users?fields=id,email,first_name
```

**Response compression:** Enable gzip/brotli compression on all text responses.
Do not apply to already-compressed content (images, video, gzip binaries).

**Connection pooling:** Database connections are expensive.
Configure a connection pool (PgBouncer or Prisma pool settings):
```
MAX_CONNECTIONS = min(cpu_count * 4, max_db_connections / instances)
```

---

### Step 6: Frontend performance

**Bundle analysis:**
```bash
npx bundlephobia [package-name]    # check package size before installing
npx @next/bundle-analyzer          # visualize your Next.js bundle
```

**Code splitting:** Never ship the entire app to users who only visit one page.
- Use dynamic imports for large components
- Use route-based splitting (Next.js does this automatically)
- Lazy-load below-the-fold content

**Image optimization:**
- Use `next/image` or equivalent (auto WebP, lazy load, correct sizing)
- Always specify `width` and `height` (prevents layout shift)
- Compress images before upload (use sharp or squoosh)

---

### Step 7: Performance budget and enforcement

Set measurable targets before starting optimization work:

| Metric | Target |
|---|---|
| API p50 response time | < 100ms |
| API p95 response time | < 500ms |
| API p99 response time | < 1000ms |
| Time to First Byte (TTFB) | < 200ms |
| Largest Contentful Paint (LCP) | < 2.5s |
| DB query time (p95) | < 50ms |
| Bundle size (initial JS) | < 200KB gzipped |

Enforce in CI: Lighthouse CI, bundle size checks, performance test thresholds.

---

### Documenting a performance improvement

After optimizing, document with before/after metrics:

```markdown
## Performance Improvement: [What was optimized]

**Date**: [date]
**Problem**: [What was slow and by how much]

**Root cause**: [Why it was slow]

**Solution**: [What was done]

**Results**:
| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| p50 response | 450ms | 35ms | 92% faster |
| DB queries/request | 47 | 3 | 94% reduction |
| DB query time | 380ms | 18ms | 95% faster |

**Index added**: CREATE INDEX CONCURRENTLY idx_orders_user_created ON orders(user_id, created_at DESC)
```

---

## What to avoid

DO NOT optimize without measuring first -- you will optimize the wrong thing.

DO NOT add caching as the first response to a slow query -- fix the query first.

DO NOT add indexes to every column "just in case" -- indexes slow writes.

DO NOT use `SELECT *` in production queries -- select only the columns you need.

DO NOT put CPU-intensive work in the main request handler -- use a background job queue.

DO NOT skip load testing before a high-traffic launch.

---

## Checklist

Before closing a performance optimization task:

- [ ] Baseline metric captured before optimization
- [ ] Root cause identified (not assumed)
- [ ] Slow queries found using DB query logs or pg_stat_statements
- [ ] EXPLAIN ANALYZE run on all slow queries
- [ ] N+1 queries fixed (query count per request verified)
- [ ] Indexes added for FKs and commonly queried columns
- [ ] Caching applied only where appropriate (and cache invalidation is correct)
- [ ] Pagination added (no unbounded list queries)
- [ ] After metric captured
- [ ] Improvement documented with before/after numbers
- [ ] New performance budget set if not already in place
- [ ] Bundle size analyzed and within budget (frontend)
- [ ] Performance test added to CI to prevent regression
