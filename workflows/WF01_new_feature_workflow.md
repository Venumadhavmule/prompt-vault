# WF01 -- New Feature Workflow

## Purpose

This is the step-by-step runbook for implementing any new feature from requirements
through to merged, tested, production-ready code. Follow every step in order.
Do not skip steps. Skipped steps cause bugs in production.

---

## Pre-conditions

- You have a clear, written requirement (ticket, issue, or design doc)
- You have access to the codebase and can run it locally
- The main branch is up to date on your machine

---

## Step-by-step

### Step 1: Read and understand the requirement

Before writing a single line of code, read the requirement completely.

Answer these questions in writing:
- What is the user trying to do?
- What is the exact input to the feature?
- What is the exact output / result?
- What is the happy path?
- What are the failure paths?
- What existing functionality does this touch?

If you cannot answer all of these from the requirement, get clarification now.
Starting to code with unclear requirements wastes more time than the clarification
would have taken.

Reference: `skills/10_mindset_read_task.md`

---

### Step 2: Break the feature into subtasks

Decompose the feature into the smallest units of work that can each be completed
and verified independently.

Required subtask categories:
- [ ] Data model changes (new tables, columns, indexes)
- [ ] Migration file
- [ ] Service/business logic layer
- [ ] API endpoint(s)
- [ ] Input validation
- [ ] Error handling
- [ ] UI components (if applicable)
- [ ] Unit tests
- [ ] Integration tests
- [ ] Documentation / CHANGELOG update

Write them as a list in your PR description (or a scratchpad) before you start.

Reference: `skills/11_mindset_break_problems.md`

---

### Step 3: Create a branch

Branch naming convention:
```
feat/short-description        # new functionality
fix/short-description         # bug fix
chore/short-description       # no behavior change
```

```bash
git checkout main
git pull origin main
git checkout -b feat/user-email-verification
```

---

### Step 4: Plan the data model

Before writing any application code:
- Identify every new table or column required
- Decide on types, constraints, nullability, and defaults
- Plan indexes for every foreign key and every query filter
- Plan the migration (additive-only -- never destructive in a single deploy)

Write the schema changes down before creating the migration file.

Reference: `skills/21_planning_database.md`

---

### Step 5: Create the Prisma migration

```bash
# Modify prisma/schema.prisma first
# Then generate the migration
npx prisma migrate dev --name add_email_verification

# Review the generated SQL before proceeding
cat prisma/migrations/[timestamp]_add_email_verification/migration.sql
```

Verify the generated SQL:
- Only adds, does not drop
- No unexpected column renames
- Indexes are present for FK columns

---

### Step 6: Define types first

Before implementing any logic, define the TypeScript types:

```
src/types/[feature].ts
```

Types to define:
- Input DTOs (what the API receives)
- Output DTOs (what the API returns)
- Internal domain objects (what the service layer works with)
- Error types specific to this feature

By defining types first, the compiler enforces consistency across all layers.

---

### Step 7: Implement the repository/data-access layer

The repository layer is the only place that talks to the database.

```
src/repositories/[feature]Repository.ts
```

- One function per query
- Functions accept typed parameters, return typed results
- No business logic here -- only data access
- No raw SQL unless Prisma cannot express the query

---

### Step 8: Implement the service layer

The service layer contains all business logic.

```
src/services/[feature]Service.ts
```

- Calls the repository, never the database directly
- Validates business rules (not just input format)
- Orchestrates multiple repository calls when needed
- Emits events or triggers side effects (emails, webhooks)
- Throws named, typed errors for failure cases

---

### Step 9: Implement the API handler

```
src/routes/[feature].ts   or   src/app/api/[feature]/route.ts
```

Handler responsibilities:
- Parse and validate request input
- Call the service
- Map service output to the API response shape
- Map service errors to HTTP status codes and error codes
- Never contain business logic

Response shape must follow the project error format. Reference: `skills/03_api_design_skill.md`

---

### Step 10: Write tests

Write tests in this order:

1. **Unit tests** for the service layer (mock the repository)
2. **Integration tests** for the API endpoints (real HTTP request, test database)
3. **Edge case tests** for known failure paths

Reference: `skills/33_execution_testing.md`

Minimum coverage:
- [ ] Happy path succeeds and returns expected shape
- [ ] Validation errors return 400 with correct error code
- [ ] Auth failures return 401/403
- [ ] Service-layer errors return correct HTTP status
- [ ] At least one database constraint failure tested (duplicate, not found)

---

### Step 11: Self-review before creating PR

Run the full check suite:

```bash
tsc --noEmit          # type check
eslint .              # lint
pnpm test             # all tests
pnpm build            # confirm build still compiles
```

Review your own diff as if you are reviewing someone else's PR:
- Read every changed file from top to bottom
- Is any error case left unhandled?
- Is there any hardcoded value that should be an env var?
- Is any `console.log` debug code left in?
- Does the API response match what was documented?

Reference: `skills/34_execution_code_review.md`

---

### Step 12: Create PR and request review

PR description must include:

```markdown
## What

[One paragraph: what is this feature?]

## Why

[Why is this needed? Link to ticket/issue]

## How

[How was it implemented? Key decisions made.]

## Testing

[How to test this. Manual steps or test command.]

## Checklist

- [ ] Types defined before implementation
- [ ] Error cases handled
- [ ] Tests written and passing
- [ ] No hardcoded secrets or IDs
- [ ] API docs updated
- [ ] CHANGELOG updated
- [ ] `tsc --noEmit` passes
- [ ] `eslint` passes
- [ ] All tests pass
```

Assign at least one reviewer. Do not merge without at least one approval.

---

## Done definition

A feature is not done until:
- All tests pass in CI
- At least one approval received
- No open must-fix comments from review
- Branch is merged to main
- Feature is deployed and smoke-tested in staging (if applicable)
- CHANGELOG entry is written
