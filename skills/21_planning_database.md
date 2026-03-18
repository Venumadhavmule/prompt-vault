# SKILL 2.2 -- How to Plan a Database Schema

## What this skill is

This skill covers how to design a database schema before writing a single migration.
A poorly designed schema is the most expensive mistake in software development because
changing it in production has a cost multiplier of 10x compared to getting it right
in planning. This skill ensures every table, relationship, column type, and index
decision is made deliberately, with the full picture in mind.

---

## When to use this skill

- At the start of any new feature that requires data persistence
- When adding new tables or significantly modifying existing ones
- When designing a new module (auth, billing, multi-tenancy, notifications)
- Before writing any Prisma schema or SQL migration
- When reviewing a PR that includes schema changes

---

## Full Guide

### Step 1: Identify entities from the feature description

An entity is a "thing" that your system needs to store and reason about.

**How to identify entities:**

1. Read the feature description and highlight all nouns
2. Ask for each noun: "Do we need to store this in the database?"
3. Ask: "Does this noun have its own lifecycle?" (created, updated, deleted independently)
4. Ask: "Do we need to query or filter by this thing independently?"

**Example from "team collaboration feature":**
Nouns: team, member, invitation, role, permission, workspace
- Team: yes (lifecycle, independent queries)
- Member: yes (junction entity between user and team)
- Invitation: yes (independent lifecycle: pending, accepted, expired)
- Role: yes IF roles are dynamic; use enum IF roles are fixed
- Permission: usually a concern of code, not DB unless building ABAC
- Workspace: yes if distinct from team

---

### Step 2: Define relationships

For each pair of entities, decide the relationship type:

| Type | Definition | Example | How to implement |
|---|---|---|---|
| One-to-One | Entity A has exactly one B, and B belongs to exactly one A | User has one Profile | FK on either side with UNIQUE constraint |
| One-to-Many | Entity A has many Bs, but each B belongs to one A | Organization has many Users | FK on the "many" side (users.org_id) |
| Many-to-Many | Entity A has many Bs, and each B has many As | Users have many Teams, Teams have many Users | Junction/join table with two FKs |

**Common relationship mistakes:**

- Storing a comma-separated list of IDs in a single column instead of a junction table
- Creating a one-to-many relationship backwards (FK on the wrong table)
- Not creating the junction table for many-to-many (you will ADD it later and regret it)

---

### Step 3: Standard columns every table must have

Every table without exception:

```sql
id          UUID        NOT NULL DEFAULT gen_random_uuid() PRIMARY KEY
created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
```

Tables that support soft delete (most business entities):
```sql
deleted_at  TIMESTAMPTZ             -- NULL = active, set = soft-deleted
```

**Why UUID over auto-increment:**
- No information leakage (users cannot enumerate resources by guessing id+1)
- Safe to generate on the client if needed
- Works across distributed/replicated systems without collision
- PostgreSQL's `gen_random_uuid()` is the correct function (v4 UUID)

---

### Step 4: Column type decisions

| Data | Correct type | Why |
|---|---|---|
| User ID, resource ID | UUID | See above |
| Money / currency amounts | INTEGER (cents) | Never use FLOAT for money. Store in smallest denomination. |
| Dates with timezone | TIMESTAMPTZ | Always store timezone. TIMESTAMP loses timezone context. |
| Boolean | BOOLEAN | Not 0/1 integers |
| Status with known values | ENUM or TEXT with CHECK constraint | Prefer TEXT + CHECK for easier migration |
| Flexible key-value metadata | JSONB | Only when schema is genuinely unknown at design time |
| Short categorized strings | TEXT with CHECK constraint | e.g., CHECK (role IN ('admin', 'member', 'viewer')) |
| Long text with no length limit | TEXT | Never use VARCHAR without a documented reason for the limit |
| Email addresses | TEXT + UNIQUE | TEXT is fine, add unique constraint, lowercase before storing |
| URLs | TEXT | No length limit needed |
| Hashed passwords | TEXT | Bcrypt output length varies (never use VARCHAR) |
| Large binary files | Do NOT store in DB | Store path/key in DB, file in object storage (S3) |

---

### Step 5: Naming conventions

**Tables:** snake_case, plural
```
users, organizations, organization_memberships, refresh_tokens, password_reset_tokens
```

**Columns:** snake_case, singular descriptive names
```
user_id, created_at, is_verified, stripe_customer_id, hashed_password
```

**Foreign keys:** always `[referenced_table_singular]_id`
```
user_id (references users), org_id (references organizations)
```

**Boolean columns:** start with `is_`, `has_`, or `can_`
```
is_verified, is_active, has_2fa, can_admin
```

**Timestamp columns:** end with `_at`
```
created_at, updated_at, deleted_at, verified_at, last_login_at
```

---

### Step 6: Index strategy

Write your index plan before writing the migration.

**Default index rules:**
1. Every primary key is automatically indexed (no action needed)
2. Every foreign key column MUST have an index (PostgreSQL does NOT auto-index FKs)
3. Every column used in a WHERE clause in common queries should have an index
4. Every column used in ORDER BY for paginated queries should have an index
5. Unique constraints automatically create a unique index

**Composite indexes:** Use when queries filter on multiple columns together.
```sql
-- If you frequently query: WHERE org_id = ? AND role = ?
CREATE INDEX idx_memberships_org_role ON organization_memberships(org_id, role);
-- Column order matters: most selective filter first
```

**Index cost:** Every index slows writes (INSERT, UPDATE, DELETE).
Only add indexes you have confirmed are needed. Do not pre-emptively index everything.

**Partial indexes:** Use for sparse data (e.g., deleted_at IS NOT NULL is rare)
```sql
CREATE INDEX idx_users_deleted ON users(deleted_at) WHERE deleted_at IS NOT NULL;
```

---

### Step 7: Migration safety rules

Every migration must be safe to run on a live database under load.

**Safe operations:**
- Adding a new nullable column
- Adding a new table
- Adding an index (use CONCURRENTLY to avoid locking)
- Dropping an index

**Unsafe operations (require extra work):**

| Operation | Problem | Safe approach |
|---|---|---|
| Adding a NOT NULL column without a default | Fails if existing rows exist | Add nullable → backfill → add NOT NULL constraint |
| Adding a NOT NULL column with a default | Can lock table during backfill | OK for small tables; use expand/contract for large tables |
| Renaming a column | Breaks any code using old name | Expand: add new column → dual-write → migrate reads → drop old |
| Changing column type | Can require full table rewrite | Add new column → copy → rename → drop old |
| Dropping a column | Immediate breakage if code still reads it | Remove code first → deploy → then drop column in next migration |
| Adding a UNIQUE constraint | Requires full table scan + lock | Create unique index CONCURRENTLY first, then add constraint using index |

**The expand/contract pattern:**
Make additive-only changes. Expand the schema to support new AND old. Deploy code
that handles both. Then contract by removing the old structure once no code uses it.

---

### Step 8: Always write the rollback migration

Every forward migration needs a paired rollback migration.

```sql
-- Forward: 20240315_add_avatar_url_to_users.sql
ALTER TABLE users ADD COLUMN avatar_url TEXT;

-- Rollback: 20240315_add_avatar_url_to_users.rollback.sql
ALTER TABLE users DROP COLUMN avatar_url;
```

You will need to rollback. Write it now, not in a crisis.

---

### Common schema mistakes and how to avoid them

| Mistake | Consequence | Correct approach |
|---|---|---|
| Storing passwords in plaintext | Catastrophic security breach | Always store bcrypt hash |
| Using FLOAT for money | Rounding errors in financial calculations | Use INTEGER (store cents) |
| Not timestamping rows | Cannot audit, cannot soft-delete, cannot debug | Always add created_at, updated_at |
| Missing FK indexes | Slow JOIN/DELETE operations on large tables | Index every FK column |
| Mega-table with 50+ columns | Impossible to maintain, slow operations | Split into focused tables |
| Using NULL for "not applicable" AND "unknown" | Ambiguous semantics | Use separate boolean columns for distinct states |
| Storing JSON for data you will later query | Can't use indexes, hard to enforce types | Use proper columns for queryable data |
| Not planning for soft delete before going live | Cannot add deleted_at safely to tables with NOT NULL constraints after data exists | Add it from day one |

---

## The Schema Planning Checklist

Before writing any migration:

- [ ] All entities identified from the feature description
- [ ] All relationships defined (1:1, 1:N, N:M) with FK columns named correctly
- [ ] Every table has: id (UUID), created_at, updated_at
- [ ] Soft delete columns (deleted_at) added to all business entity tables
- [ ] Column types are correct: no FLOAT for money, no TIMESTAMP without zone, no VARCHAR where TEXT is fine
- [ ] All FK columns have explicit indexes
- [ ] All commonly queried columns have planned indexes
- [ ] Unique constraints are identified and planned
- [ ] NOT NULL vs nullable decision made for every column with documented reasoning
- [ ] Forward migration written
- [ ] Rollback migration written alongside the forward migration
- [ ] Migration is safe to run live (no table locks, additive only or using expand/contract)
- [ ] If existing rows need backfilling, a separate backfill migration or script is written
- [ ] Naming follows: snake_case tables (plural), snake_case columns, [table]_id for FKs
- [ ] No data stored in DB that belongs in object storage (images, documents, etc.)
