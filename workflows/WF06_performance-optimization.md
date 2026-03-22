# WF06 -- Performance Optimization Workflow

## Purpose

Step-by-step runbook for diagnosing and fixing performance problems. Performance
work done without measurement is guesswork. This workflow forces measurement before
action and verification after every change.

---

## Core rule

Never optimize what you have not measured.
Never deploy a "performance fix" without measuring the before and after.

---

## Step 1: Define the problem precisely

Before investigating, write down:
- What is slow? (specific endpoint, query, page load, background job?)
- How slow? (what is the current P50/P95/P99 response time?)
- What is acceptable? (what is the target?)
- When did it become slow? (always, or a regression after a specific change?)

If you cannot answer these questions, instrument first (see Step 2).

---

## Step 2: Instrument and measure first

**For API endpoints:**
- Check your APM tool (Datadog, New Relic, etc.) for P50/P95/P99 latencies
- Check per-endpoint breakdown to isolate which endpoint is slow
- Look at the "top slow spans" to see the specific operations consuming time

**For database queries:**
- Enable Prisma query logging:
  ```typescript
  const prisma = new PrismaClient({
    log: ['query', 'info', 'warn'],
  })
  ```
- Or use `$queryRawUnsafe` with EXPLAIN ANALYZE for a specific query:
  ```sql
  EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';
  ```

**For frontend:**
- Run Lighthouse audit in Chrome DevTools
- Check Core Web Vitals (LCP, CLS, FID) in Chrome DevTools Performance tab
- Use `next/bundle-analyzer` to visualize bundle size

**For background jobs:**
- Add timing logs at the start and end of each job
- Track wall time and database time separately

---

## Step 3: Identify the bottleneck category

Map the measured data to a bottleneck type:

| Symptom | Likely cause |
|---------|-------------|
| Query taking > 100ms | Missing index, full table scan, bad query plan |
| Many fast queries (N+1) | Missing eager loading / join |
| API is slow but DB is fast | CPU-bound computation, serialization, external service call |
| Memory keeps growing | Memory leak, unbounded in-memory cache, missing pagination |
| Slow after data volume grew | Query that was O(n) is now expensive; needs index or redesign |
| Slow only at peak load | Connection pool exhaustion, lock contention |
| Large page bundle | Unoptimized imports, missing code splitting, large dependency |

---

## Step 4: Fix the bottleneck

Work from most impactful to least impactful.

### Database optimization sequence:

1. **Add a missing index** (biggest wins, lowest risk)
   ```sql
   -- Check if an index would help
   EXPLAIN ANALYZE [your query here];
   -- Look for "Seq Scan" on large tables -- that is where indexes help
   
   -- Add the index concurrently (does not lock the table)
   CREATE INDEX CONCURRENTLY idx_users_email ON users(email);
   ```

2. **Fix N+1 queries** (use `include` in Prisma)
   ```typescript
   // BAD: N+1 -- queries for each post's author separately
   const posts = await prisma.post.findMany();
   for (const post of posts) {
     const author = await prisma.user.findUnique({ where: { id: post.authorId } });
   }
   
   // GOOD: Single query with join
   const posts = await prisma.post.findMany({
     include: { author: true },
   });
   ```

3. **Add caching** (for reads that are expensive and rarely change)
   - Redis for results that take > 50ms to generate
   - Cache the output of a query that runs on every request
   - Set TTL appropriate to how often the data changes
   - Cache invalidation must be implemented alongside cache introduction

4. **Paginate large result sets** (for queries that return unbounded rows)
   - Cursor-based pagination for sorted feeds
   - Offset pagination for admin pages with random access
   - Default page size of 20-50 items

5. **Optimize query design** (rewrite queries that cannot be sped up by indexes)
   - Replace subqueries with JOINs
   - Use CTEs for readability on complex queries
   - Move heavy aggregations to a materialized view or summary table

### Application layer optimization:

- Move heavy computation out of the hot path into a background job
- Use streaming for large responses
- Use connection pooling (PgBouncer) if connection count is a bottleneck
- Move external API calls behind a cache or run them async

### Frontend optimization:

- Dynamic imports for large components: `const Chart = dynamic(() => import('./Chart'), { ssr: false })`
- Image optimization: use `next/image` or equivalent
- Remove unused dependencies
- Split large bundles by route

---

## Step 5: Measure after the change

After every change:
1. Measure the same metric you measured in Step 1
2. Calculate the improvement: (before - after) / before × 100%
3. Confirm no regression was introduced in adjacent functionality

**Do not skip this step.** "It feels faster" is not a measurement.

---

## Step 6: Deploy and verify in production

- Deploy the single change
- Watch the performance metrics for 15 minutes post-deploy
- Confirm P95 latency improved for the target endpoint
- Confirm no error rate increase

---

## Step 7: Document the finding

Add a comment in the code for non-obvious optimizations:
```typescript
// This query uses composite index (user_id, created_at DESC) to avoid
// a sequential scan on the events table which has 50M+ rows.
// Adding a filter on any column NOT in this index will cause a full scan.
const events = await prisma.event.findMany({
  where: { userId },
  orderBy: { createdAt: 'desc' },
  take: 20,
});
```

---

## Checklist

- [ ] Slow operation measured (P50/P95/P99 or total duration)
- [ ] Target performance defined (what is "fixed"?)
- [ ] Bottleneck category identified (DB, N+1, cache, bundle, etc.)
- [ ] Fix targets the root cause, not the symptom
- [ ] After fix: same metric re-measured to confirm improvement
- [ ] No regression introduced in adjacent functionality
- [ ] Tests still pass after change
- [ ] Non-obvious optimization choices documented in comments
- [ ] If an index was added: EXPLAIN ANALYZE confirms it is being used
- [ ] If a cache was added: cache invalidation logic is implemented
