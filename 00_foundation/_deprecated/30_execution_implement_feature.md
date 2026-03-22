# SKILL 3.1 -- How to Implement Any Feature Step by Step

## What this skill is

This skill gives the mandatory execution order and discipline for implementing any
software feature. It addresses the most common engineering failure mode: writing code
in a disorganized order that produces working-looking output with hidden bugs, untested
logic, and architectural violations. Following this skill guarantees that every feature
is built on a solid foundation, tested at each layer, and ready for production.

---

## When to use this skill

- When starting the implementation phase of any feature after planning is complete
- When building any new module (auth, billing, notifications, file upload, etc.)
- When an AI agent begins generating code for a defined feature
- When onboarding onto an existing codebase and adding to it

---

## Full Guide

### The mandatory execution order

This order is non-negotiable. Every deviation creates rework.

```
1.  Types / Interfaces
2.  Validation Schemas (Zod or equivalent)
3.  Database Migration (forward + rollback)
4.  Repository / Data Access Layer
5.  Service Layer
6.  API Layer (Controller / Route Handler)
7.  Middleware (auth guard, rate limit, validation wired up)
8.  UI Components (if applicable)
9.  UI Pages / Screens (if applicable)
10. Tests (alongside each layer)
11. Documentation updates
```

---

### Step 1: Types and Interfaces

Before any code with logic, define the shape of your data.

What to define:
- Domain entity types (the shape of data as stored and used in the system)
- Input types (what the API receives)
- Output types (what the API returns)
- Internal function parameter and return types

Rules:
- Types are in their own file: `types/user.types.ts`
- No logic in type files -- only type definitions
- Use `interface` for objects that may be extended; use `type` for unions, intersections, and aliases
- Mark optional fields with `?` only when they are genuinely optional
- Use strict types -- avoid `any`. If the type is unknown input, use `unknown` and narrow it explicitly.

---

### Step 2: Validation Schemas

Write Zod schemas for every input that crosses a system boundary (API requests,
form submissions, environment variables, queue messages).

Rules:
- Zod schema lives next to the type it validates: `schemas/user.schema.ts`
- Export both the schema AND the inferred TypeScript type from the same file
- Parse at the boundary, not deep in business logic
- Validation failures must produce specific field-level error messages, not generic ones
- `.transform()` is allowed for normalization (e.g., `.toLowerCase()` on email)
- Do not use `.optional()` as a shortcut when the API contract says required

---

### Step 3: Database Migration

Write the forward migration first, then the rollback immediately after.

Rules:
- Never manually edit a migration after it has been run, even in development
- Test the migration on a clean schema
- Test the ROLLBACK migration immediately after the forward
- Use `gen_random_uuid()` for UUID primary keys
- Add indexes inline with column definitions for small tables, use CREATE INDEX CONCURRENTLY for
  adding indexes to existing large tables in production

---

### Step 4: Repository / Data Access Layer

The repository is the only place in the application that talks to the database.

**What belongs in the repository:**
- All database queries (SELECT, INSERT, UPDATE, DELETE)
- Soft delete filtering (`WHERE deleted_at IS NULL` applied here, not in service)
- Pagination logic
- Transaction coordination

**What does NOT belong in the repository:**
- Business rules ("user cannot update profile if email is not verified")
- HTTP concepts (no req/res objects)
- Error formatting for the client
- Calls to external services

**Repository method naming:**
```
findById(id)           -- returns one or null
findMany(params)       -- returns list with pagination meta
create(data)           -- returns created entity
update(id, data)       -- returns updated entity
delete(id)             -- returns void or deleted entity
findByEmail(email)     -- custom finders follow findBy[Field] pattern
```

---

### Step 5: Service Layer

The service layer contains ALL business logic. It orchestrates repository calls and
external service calls to implement a business operation.

**What belongs in the service:**
- Validation logic that depends on existing data (check email uniqueness)
- Authorization logic (can this user perform this action?)
- Business rules (trial period logic, rate limiting logic, pricing rules)
- Multi-step operations (create user + send email + log audit)
- Error throwing (throw a domain error when a rule is violated)

**What does NOT belong in the service:**
- Direct database queries (use the repository)
- HTTP concepts (no req/res/status codes)
- UI concerns

**Service rules:**
- Services are injectable (constructor-injected dependencies for testability)
- Services throw typed errors (use AppError subclasses from the error system)
- Services are synchronous in their logic -- no fire-and-forget without explicit acknowledgment
- Services do not catch and silently swallow errors -- let them propagate

---

### Step 6: API Layer

The API layer handles only HTTP concerns. It delegates all logic to the service layer.

**What belongs in the API layer:**
- Parsing request body, params, and query string
- Calling the validation schema (parse → get typed data → call service)
- Calling the appropriate service method
- Formatting the response (success shape and status code)
- Catching service errors and mapping to HTTP response (via global error handler)

**What does NOT belong in the API layer:**
- Business logic (no "if user.role === 'admin'" in the route handler)
- Direct database queries
- Complex data transformation

**Route handler pattern:**
```
handler(req, res) {
  const input = schema.parse(req.body)       // validate
  const result = await service.method(input) // delegate
  return res.status(201).json({ data: result }) // respond
}
```

---

### Step 7: Naming conventions (enforced rules)

| Item | Convention | Example |
|---|---|---|
| Functions | camelCase, verb-first | `getUserById`, `sendResetEmail`, `calculateProration` |
| Variables | camelCase, descriptive | `userId`, `hashedPassword`, `expiresAt` |
| Constants | SCREAMING_SNAKE_CASE | `MAX_LOGIN_ATTEMPTS`, `JWT_EXPIRY_MINUTES` |
| Classes | PascalCase | `UserService`, `PaymentsRepository` |
| Files | kebab-case | `user-service.ts`, `auth-middleware.ts` |
| Types / Interfaces | PascalCase | `UserProfile`, `CreateUserInput` |
| Booleans | `is`/`has`/`can` prefix | `isVerified`, `hasSubscription`, `canDelete` |
| Event handlers | `handle` prefix | `handleStripeWebhook`, `handlePasswordReset` |
| Repository methods | CRUD verbs | `findById`, `create`, `update`, `delete`, `findMany` |

---

### Step 8: Input validation at every layer

Validate at the boundary (API layer). Re-validate at critical operations.

```
API Layer    → Zod schema validates raw HTTP input (shape, types, formats)
Service Layer → Business rule validation (does the entity exist? does the user have access?)
Repository   → DB constraints are the last line of defense (NOT NULL, UNIQUE, FK)
```

Never validate only at one layer and trust everything below it. Each layer defends itself.

---

### Step 9: Error handling

Use typed errors throughout. The global error handler processes all of them consistently.

```
throw new ValidationError('Email is invalid', [{ field: 'email', message: '...' }])
throw new NotFoundError('User not found')
throw new ForbiddenError('You do not have permission to delete this resource')
throw new ConflictError('Email address is already registered')
```

The global error handler:
1. Catches all thrown errors in route handlers
2. If it is an AppError (operational): formats it using the standard error shape, returns the correct status
3. If it is unknown (programmer error): logs the full error server-side, returns a generic 500 with requestId only

NEVER leak: stack traces, DB error messages, file paths, internal state to the client.

---

### Step 10: The self-review checklist for every file

Before moving to the next file, run this review on every file you write:

- [ ] No TypeScript `any` types
- [ ] All async functions have try/catch or propagate errors to a global handler
- [ ] All external inputs are validated before use
- [ ] No business logic in the API layer
- [ ] No HTTP concepts in the service layer
- [ ] No direct DB calls outside the repository layer
- [ ] All error paths are handled (not just the happy path)
- [ ] Function names are verbs and describe what the function does
- [ ] No hardcoded secrets, URLs, or magic numbers (use constants or env vars)
- [ ] Tests are written or updated for this file

---

## What to avoid

DO NOT start with the UI before the data model and API are finalized.

DO NOT put business logic in the route handler.

DO NOT put HTTP concepts (req, res, status codes) in the service layer.

DO NOT query the database directly from service or API code. Use the repository.

DO NOT use `any` type. Use `unknown` and narrow it explicitly.

DO NOT skip the rollback migration.

DO NOT write tests only at the end. Write unit tests as you finish each service method.

DO NOT silently swallow errors with empty catch blocks.

DO NOT return error details that leak internal implementation.

---

## Checklist

Before marking any feature implementation as complete:

- [ ] Types and interfaces defined
- [ ] Validation schemas written for all inputs
- [ ] Forward and rollback migrations written and tested
- [ ] Repository layer implemented with correct method names
- [ ] Service layer contains all business logic
- [ ] API layer delegates to service with no business logic
- [ ] Auth and rate limit middleware applied to all relevant routes
- [ ] All error cases handled and typed errors thrown
- [ ] Global error handler processes errors without leaking internals
- [ ] Unit tests written for service layer
- [ ] Integration tests written for API routes
- [ ] No `any` types in the codebase
- [ ] No hardcoded secrets or magic numbers
- [ ] .env.example updated if new env vars added
- [ ] OpenAPI documentation updated
