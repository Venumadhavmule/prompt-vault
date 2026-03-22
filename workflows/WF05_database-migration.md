# WF05 -- Database Migration Workflow

## Purpose

Step-by-step runbook for making any schema change safely in a production system.
Migrations that drop columns, rename things, or change constraints can cause
immediate downtime without the correct sequence. This workflow ensures zero-downtime
schema changes.

---

## Pre-conditions

- Prisma is the migration tool (adjust commands if using another tool)
- The application is deployed and running in production
- You have the ability to run migrations against production (IAM role or connection)

---

## Core principle: Expand/Contract pattern

Never change AND remove in the same deployment.

Every breaking schema change requires THREE deployments:
1. **Expand**: Add the new structure while keeping the old one
2. **Migrate**: Move/copy data from old to new (if needed)
3. **Contract**: Remove the old structure after the application no longer uses it

---

## Step 1: Classify the change

Determine whether your migration is safe or breaking:

**Safe (can deploy in one step):**
- Adding a new nullable column
- Adding a new table
- Adding a new index
- Widening a column type (VARCHAR(50) → VARCHAR(255))

**Breaking (requires expand/contract):**
- Renaming a column
- Dropping a column
- Adding a NOT NULL constraint to an existing column
- Changing a column type (narrowing or fundamentally different)
- Renaming a table
- Changing a foreign key relationship

---

## Step 2: Plan the migration sequence

For safe migrations:
```
1. Write migration
2. Test in dev
3. Test in staging (with production-like data volume)
4. Deploy migration to production
5. Deploy application change
```

For breaking migrations (expand/contract):
```
Deployment 1:
  - Add new column/table (the "expand")
  - Deploy application that writes to BOTH old and new column

Deployment 2 (optional, if data migration needed):
  - Run data migration script to backfill new column from old
  - Verify backfill is complete

Deployment 3:
  - Deploy application that reads from new column only
  - Stop writing to old column

Deployment 4 (the "contract"):
  - Drop the old column/table
```

---

## Step 3: Write the migration

```bash
# In development, modify prisma/schema.prisma first
# Then generate the migration
npx prisma migrate dev --name describe_the_change
```

Review the generated SQL **in full** before proceeding:
```bash
cat prisma/migrations/[timestamp]_describe_the_change/migration.sql
```

Check for:
- [ ] No `DROP COLUMN` or `DROP TABLE` unless this is the contract phase
- [ ] `NOT NULL` constraint only on columns that have a `DEFAULT` or where all rows will be backfilled
- [ ] No `ALTER TABLE ... RENAME` unless it is preceded by a deploy that handles both names
- [ ] Index creation uses `CREATE INDEX CONCURRENTLY` for large tables (if applicable)

---

## Step 4: Test the migration on development

```bash
# Apply migration
npx prisma migrate dev

# Verify the schema
npx prisma studio
# or
npx prisma db pull  # to see if schema.prisma matches the applied state

# Run all tests
pnpm test
```

---

## Step 5: Test on staging

Apply the migration to staging with a snapshot of production data (or representative volume).

```bash
# On staging
npx prisma migrate deploy
```

Verify:
- [ ] Migration completes without error
- [ ] Application starts correctly with the new schema
- [ ] All existing functionality works (run smoke tests)
- [ ] Query performance is not degraded (check EXPLAIN ANALYZE for the affected queries)

---

## Step 6: Deploy to production

Sequence for production deployment:

```bash
# 1. Apply migration (before deploying new application code)
npx prisma migrate deploy

# 2. Verify migration applied
npx prisma migrate status

# 3. Deploy application code
[your deployment mechanism]

# 4. Verify application started and is healthy
[check health endpoint]
```

For rolling deployments (multiple instances):
- Apply the migration before rolling out the new application code
- The new schema must be backward compatible with the OLD application code
  (this is why the expand step keeps the old column -- old code still works during rollout)

---

## Step 7: Post-migration verification

- [ ] `npx prisma migrate status` shows no pending migrations
- [ ] Health check endpoint is green
- [ ] Application logs show no database errors in the 5 minutes after deploy
- [ ] Run critical user flows manually to confirm behavior is correct
- [ ] Verify that query performance has not regressed

---

## Rollback

Prisma does not support automatic rollback. If you need to reverse:

**For safe migrations (added a nullable column):**
```bash
# Write a new migration to drop it
npx prisma migrate dev --name rollback_column_name
```

**For breaking migrations:**
- If the contract phase has not been applied: the old column/structure is still there.
  Revert the application code. No schema change needed.
- If the contract phase was applied and data was lost: restore from backup.
  This is why you keep backups and verify before the contract step.

---

## Checklist

- [ ] Change classified (safe vs breaking)
- [ ] If breaking: expand/contract sequence planned as separate deployments
- [ ] Migration SQL reviewed manually (no unintended DROP or RENAME)
- [ ] Migration tested in development environment
- [ ] Migration tested in staging with production-like data volume
- [ ] Query plan checked for index usage (EXPLAIN ANALYZE) in staging
- [ ] Migration applied to production BEFORE new application code deployed
- [ ] Post-migration verification completed
- [ ] Application logs clean for 5 minutes after deploy
