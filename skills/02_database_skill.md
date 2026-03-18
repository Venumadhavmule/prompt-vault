# Database Skill -- Instruction Set

> Load this skill at the START of any task involving database schema design,
> migrations, queries, ORMs, data modeling, repositories, or data access patterns.
> Do not proceed without reading all sections.

---

## Prime Directive

The database is the most permanent layer of your system.
Schema decisions are extremely costly to reverse.
Query decisions directly determine user-facing performance.
You design before you code. You migrate before you deploy.
You measure before you optimize.

---

## 1. Pre-Work: Identify the Task Type

Before writing anything, classify the task:

| Task Type | Key Concerns |
|---|---|
| New schema design | Normalization, indexes, constraints, future growth |
| Adding a column | Nullable vs NOT NULL, migration safety, backfill strategy |
| Adding a table | Relationships, FK constraints, cascade rules |
| Query optimization | EXPLAIN ANALYZE first, index strategy, N+1 detection |
| Migration | Forward + rollback both required, zero-downtime strategy |
| Data seeding | Idempotency (safe to run multiple times), no prod data in seed |
| Multi-tenancy | Row-level isolation vs schema-per-tenant vs DB-per-tenant |

State which type applies before starting.

---

## 2. Schema Design Rules -- Non-Negotiable

### Naming

All table and column names: `snake_case`, plural for tables.
```
CORRECT: users, refresh_tokens, organization_members, subscription_invoices
WRONG:   User, refreshToken, OrganizationMember
```

Every table MUST have:
- `id` (UUID primary key, default `gen_random_uuid()`) -- NEVER use auto-increment integers for user-facing IDs
- `created_at` (timestamptz, NOT NULL, default `now()`)
- `updated_at` (timestamptz, NOT NULL, default `now()`, auto-updated via trigger or ORM hook)

Use `timestamptz` (timestamp with time zone), NEVER `timestamp` without zone.

### Nullable Columns

NEVER make a column nullable unless null genuinely means "unknown" or "not yet set".
Nullable columns are a major source of bugs (null checks everywhere) and query confusion.
If a column has a logical default, use the default. Only reach for nullable as a last resort.

### Soft Deletes

Use soft deletes (`deleted_at timestamptz`) ONLY when:
- You need audit trail of deleted records
- Records are referenced by other tables and hard deletion would cause FK violations
- You need "undo" functionality

When using soft deletes:
- Add `WHERE deleted_at IS NULL` to ALL queries by default (use a repository base class or ORM scope)
- Add an index on `deleted_at`
- Periodically hard-delete old soft-deleted records via a maintenance job

Do NOT use soft deletes for everything by default. Hard delete is cleaner and simpler when not needed.

### Foreign Keys

ALWAYS define foreign key constraints at the DB level, not just at the ORM level.
DB-level constraints are the last line of defense against data corruption.

Define cascade behavior explicitly on every FK:
```
ON DELETE CASCADE    -- delete child when parent is deleted (use carefully)
ON DELETE RESTRICT   -- prevent parent deletion if children exist (safest default)
ON DELETE SET NULL   -- set FK to null when parent is deleted
ON DELETE NO ACTION  -- equivalent to RESTRICT, checked at end of transaction
```

---

## 3. Index Strategy -- Know This Before Writing Queries

### Always Index

- Every foreign key column (PostgreSQL does NOT do this automatically)
- Every column used in `WHERE` clauses on large tables
- Every column used in `ORDER BY` if the table is large
- Every column used in `JOIN` conditions (beyond primary keys)
- Unique constraints (these create indexes automatically)

### Composite Index Rules

Column order in composite indexes matters. Put the highest-cardinality, most-selective column FIRST.
```
CORRECT: CREATE INDEX ON orders (user_id, status, created_at) -- filtering by user first
WRONG:   CREATE INDEX ON orders (created_at, user_id, status) -- created_at has lowest selectivity as a filter
```

### Partial Indexes

Use partial indexes for filtered queries on non-nullable columns:
```sql
CREATE INDEX idx_users_unverified ON users (created_at) WHERE email_verified_at IS NULL;
```
Much smaller and faster than a full index when the condition matches a minority of rows.

### When NOT to Index

- Very small tables (< 1000 rows): full scans are faster than index lookups
- Columns rarely used in WHERE/JOIN/ORDER BY
- Tables with extremely high write volume (index maintenance has write cost)

---

## 4. Repository Pattern Rules

The repository is the ONLY place that talks to the database.
Services call repositories. Services do NOT write raw queries.
Controllers call services. Controllers do NOT call repositories directly.

### Repository Contract

Every repository MUST expose:
```
findById(id)         -> T | null
findMany(filters)    -> { items: T[], total: number, nextCursor?: string }
create(data)         -> T
update(id, data)     -> T
delete(id)           -> void
exists(filters)      -> boolean
```

### Repository Rules

- NEVER return raw database errors to callers -- wrap in typed domain errors
- NEVER put business logic inside a repository (no "if user is admin" inside a query)
- ALWAYS return domain types, not raw ORM model types to callers above service layer
- ALWAYS include pagination on `findMany` -- never return unbounded lists
- ALWAYS scope queries to the current user/org to prevent cross-tenant data leakage

---

## 5. Migration Rules -- Follow Every Time

### The Two-File Rule

Every schema change requires TWO files:
1. The forward migration (applies the change)
2. The rollback migration (undoes the change completely)

You NEVER write a migration without also writing its rollback.

### Migration Safety Checklist

Before writing the migration, answer:

| Question | Why It Matters |
|---|---|
| Will this lock the table? | ALTER TABLE on large tables causes full table lock in PostgreSQL |
| Is any new column NOT NULL without a default? | Will fail on tables with existing rows |
| Does this rename a column or table? | Will break ORM queries until code is deployed simultaneously |
| Does this drop a column or table? | Irreversible data loss -- must be the LAST step of a multi-deploy process |

### Zero-Downtime Migration Strategy

For production tables with existing data, use this sequence:

**Adding a NOT NULL column with no default:**
```
Step 1: Add column as nullable (immediate, no lock)
Step 2: Backfill existing rows in batches (no lock, can run while app is live)
Step 3: Add NOT NULL constraint (locks only to check -- fast if backfill is complete)
```

**Renaming a column:**
```
Step 1: Add new column with new name
Step 2: Deploy code that writes to BOTH old and new columns
Step 3: Backfill new column from old column
Step 4: Deploy code that reads from new column only
Step 5: Drop old column
```

**Never do a rename in a single migration on a live system.**

### Migration File Naming

```
{timestamp}_{action}_{subject}.sql
Examples:
  20240315_create_users.sql
  20240316_add_stripe_customer_id_to_users.sql
  20240317_create_subscriptions.sql
```

---

## 6. Query Safety Rules

### Always Use Parameterized Queries

NEVER concatenate user input into SQL strings.
```
WRONG:  `SELECT * FROM users WHERE email = '${email}'`
CORRECT: `SELECT * FROM users WHERE email = $1` with params: [email]
```

This applies equally to ORM query builders. Always use `where: { field: value }`, never raw string interpolation.

### Pagination Rules

NEVER return unbounded results (`SELECT * FROM orders` with no LIMIT).
ALWAYS implement one of:
- Offset pagination: `LIMIT n OFFSET n*page` (simple, but slow on large offsets)
- Cursor pagination: `WHERE id > $cursor LIMIT n` (fast at any depth, preferred for large datasets)

Default page size: 20. Maximum page size: 100 (enforce server-side, never trust client-provided limit).

Cursor-based pagination is REQUIRED when:
- The table has more than 100k rows
- Real-time updates could cause page drift
- The user needs to "load more" (infinite scroll)

### N+1 Query Detection

Before shipping any endpoint that returns a list:
- Trace the ORM queries or log SQL in dev
- If you see N queries for N items (one query per list item), fix with JOIN or eager loading
- ORM rule: use `include`/`with` to eager-load relations, never lazy-load in a loop

---

## 7. Transaction Rules

Use a transaction anytime more than one write must succeed or fail together.

```
Create user + Create default org + Create membership = one transaction
Deduct balance + Create transaction record = one transaction
Update subscription + Write audit log = one transaction
```

Transaction rules:
- Keep transactions SHORT -- long-held locks cause timeouts and deadlocks
- NEVER make network calls (HTTP, email, queue) inside a database transaction
- NEVER make transactions that span multiple request cycles
- On error inside a transaction: rollback and rethrow the error

---

## 8. Multi-Tenancy Data Isolation

If the app has multiple organizations/tenants, EVERY query MUST be scoped to the current org.

Approach: Row-level isolation (single DB, `org_id` column on every tenant-scoped table)

Rules:
- `org_id` must be a NOT NULL FK on every table that belongs to an org
- ALL repository methods must accept and enforce `orgId` parameter
- Middleware must resolve and validate the current org before any route handler runs
- NEVER trust `orgId` from the request body -- resolve it from the authenticated user's context

Test: Always write a test that proves User A from Org X CANNOT access data from Org Y.

---

## 9. ORM vs Raw SQL Decision Rule

| Situation | Use |
|---|---|
| Standard CRUD operations | ORM (Prisma / Drizzle) |
| Complex joins, aggregations, CTEs | Raw SQL with parameterized queries |
| Migrations | Raw SQL (never let ORM auto-migrate production) |
| Reporting / analytics queries | Raw SQL |
| Performance-critical hot path (< 10ms target) | Raw SQL or ORM with profiling |
| Batch inserts (1000+ rows) | Raw SQL with `COPY` or bulk insert |

Never auto-sync ORM schema to production database. Always use explicit migration files.

---

## 10. Connection Pool Configuration

For production PostgreSQL:
```
min connections:     2 (always keep alive)
max connections:     10 per app instance (tune based on DB server limit)
idle timeout:        10 seconds (return idle connections to pool)
connection timeout:  5 seconds (fail fast, don't queue forever)
statement timeout:   30 seconds (kill runaway queries, prevent DB overload)
```

In a serverless environment (Vercel, Lambda, Cloudflare Workers):
- Use a connection pooler like PgBouncer or Neon's connection pooling
- Or use a serverless-compatible ORM like Drizzle in HTTP mode

---

## 11. Database Health Check

Every application MUST implement a health check that verifies DB connectivity:
```
GET /health
Response: { db: 'connected' | 'disconnected', latency_ms: number }
```

The health check must run a real query (even `SELECT 1`) to confirm connectivity.
It must NOT expose connection strings, credentials, or server information.

---

## 12. What to NEVER Do

- Never run raw migrations in production without first testing on a staging DB copy
- Never use `SELECT *` -- always select specific columns
- Never store JSON blobs as the primary data store for queryable fields
- Never ignore database errors -- always handle them and return typed errors
- Never expose raw ORM errors to clients (they can leak schema information)
- Never write business logic directly in SQL unless it's a specific performance optimization
- Never seed test data into production
- Never put credentials in migration files

---

## 13. Cross-References

- Error types: `DatabaseError`, `NotFoundError` -> see `00_error_system`
- Schema for auth tables -> see `01_authentication_skill`
- Connection string env vars -> see `00_environment_config`
- API pagination patterns -> see `03_api_design_skill`
- Testing with real DB -> see `05_testing_skill`
