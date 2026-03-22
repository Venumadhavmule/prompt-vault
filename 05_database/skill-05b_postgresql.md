# SKILL 05-B — PostgreSQL in Production: Setup, Indexing, and Query Optimization
Category: Database
Applies to: PostgreSQL, pgAdmin, Supabase, Neon, RDS PostgreSQL

## What this skill covers
Running PostgreSQL in production requires more than creating tables and running queries. This skill covers index strategy (btree, GIN, partial, composite), query analysis with `EXPLAIN ANALYZE`, connection configuration, common performance anti-patterns, and essential production settings (WAL, autovacuum, connection limits) that most tutorials omit.

## When to activate this skill
- Diagnosing a slow PostgreSQL query
- Designing the index strategy for a new table
- Configuring a PostgreSQL instance for production
- Experiencing connection exhaustion or autovacuum-related bloat

## Core principles
1. **Index for your queries, not your tables.** Index the columns you filter and sort by in your most critical queries. Extra indexes slow down writes.
2. **`EXPLAIN ANALYZE` before optimizing.** Never change an index or query structure based on intuition — read the query plan first.
3. **Autovacuum must be monitored.** Table bloat from dead tuples slows all queries. Autovacuum settings need tuning for high-write tables.
4. **Connection count is a hard resource.** PostgreSQL creates a process per connection. At 200+ connections, use a pooler (PgBouncer, Supabase Pooler) in transaction mode.
5. **Avoid `SELECT *` in application code.** Fetching unused columns wastes I/O and prevents index-only scans.

## Step-by-step guide
1. **Analyze slow queries:** run `EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT ...`. Look for `Seq Scan` on large tables — these indicate missing indexes. Look for `Sort` operations on unsorted data.
2. **Choose the right index type:**
   - `BTREE` (default): equality, range, ordering — `WHERE email = ?`, `WHERE created_at > ?`
   - `GIN`: full-text search (`tsvector`), JSONB contains queries (`@>`)
   - `GiST`: geographic data, range types
   - `Partial index`: `CREATE INDEX ON orders(user_id) WHERE status = 'pending'` — smaller, faster for filters
   - `Composite index`: follow the order of columns in `WHERE` clauses — most selective first
3. **Create composite indexes intelligently:** index columns in the order they narrow results. If most queries filter by `user_id` then sort by `created_at`, the index is `(user_id, created_at)`.
4. **Enable `pg_stat_statements`** to find the top-N slowest queries over time: `SELECT query, mean_exec_time FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 10`.
5. **Configure `max_connections` and use PgBouncer:** set `max_connections = 100` in `postgresql.conf`, configure PgBouncer with `pool_mode = transaction` for connection pooling.
6. **Tune autovacuum for write-heavy tables:** lower `autovacuum_vacuum_scale_factor` to `0.01` and `autovacuum_analyze_scale_factor` to `0.005` on large tables.
7. **Use `RETURNING` in mutations** to avoid a separate `SELECT` after `INSERT` or `UPDATE`.

## The right way vs the wrong way

| Wrong | Right |
|-------|-------|
| `SELECT *` in production queries | Select only the columns needed; allows index-only scans |
| Index every column "just in case" | Index based on actual query plans from `EXPLAIN ANALYZE` |
| `WHERE LOWER(email) = $1` without a functional index | `CREATE INDEX ON users (LOWER(email))` — functional index on the expression |
| `SELECT id FROM users WHERE email LIKE '%@example.com'` | Leading wildcard prevents index use — use `pg_trgm` with GIN index for contains searches |
| 500 app server connections directly to PostgreSQL | PgBouncer in transaction mode with `max_connections = 100` in PG |

## Stack-specific notes
**Node.js (Prisma/Drizzle):** Use `EXPLAIN` via raw query access: Prisma `$queryRaw`, Drizzle `.toSQL()`. Enable `queryLog` in development to see all queries.
**Python (SQLAlchemy):** `echo=True` on the engine logs queries in development. Use `text("EXPLAIN ANALYZE ...")` for plan analysis.
**Supabase/Neon:** Both provide a query editor with `EXPLAIN ANALYZE` support. Supabase has a performance advisor in the dashboard. Neon supports connection pooling via its proxy.

## Common mistakes
1. **Forgetting to `ANALYZE` after large batch imports.** PostgreSQL's query planner uses statistics. After bulk loading data, run `ANALYZE tablename` to update statistics.
2. **Using `OFFSET` for pagination at scale.** `OFFSET 10000 LIMIT 20` scans 10,020 rows. Use keyset pagination: `WHERE id > $last_seen_id LIMIT 20`.
3. **`ORDER BY` on a non-indexed column in a large result set.** PostgreSQL performs a full table scan + sort. Add an index on the sort column.
4. **Treating JSON columns like a document store.** JSONB queries can be indexed with GIN, but complex JSON traversal is slower than normalized relational queries. Normalize frequently queried fields.

## Checklist
- [ ] `EXPLAIN (ANALYZE, BUFFERS)` run on all slow queries before adding indexes
- [ ] Index type chosen based on query pattern (BTREE / GIN / partial / composite)
- [ ] `pg_stat_statements` enabled for ongoing slow query monitoring
- [ ] `SELECT *` replaced with explicit column lists in all application queries
- [ ] PgBouncer or platform pooler configured for connection pooling
- [ ] `max_connections` set to match available memory and pooler configuration
- [ ] Autovacuum settings tuned for high-write tables
- [ ] Keyset pagination used instead of `OFFSET` for large datasets
- [ ] Functional indexes created for expressions used in `WHERE` (e.g. `LOWER()`)
- [ ] `RETURNING` clause used for mutations to avoid a follow-up `SELECT`
