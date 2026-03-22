# SKILL 04-A — Universal Backend Project Structure
Category: Backend Development
Applies to: Node.js (Express/Fastify/Hono), Python (FastAPI/Django), Go; any REST or GraphQL server

## What this skill covers
Regardless of the framework or language, a production backend follows the same structural principles: separate concerns by layer, isolate business logic from transport, keep framework code at the edges, and make the domain independently testable. This skill defines the universal folder layout and layering model that applies before any language or framework specifics are considered.

## When to activate this skill
- Starting a new backend service from scratch
- Reviewing or refactoring a messy backend codebase
- Deciding where a new piece of code should live
- Onboarding a team to a standard structure

## Core principles
1. **Separate transport from domain.** HTTP-specific code (request parsing, status codes) never touches business logic.
2. **Routes declare, handlers orchestrate, services execute.** Each layer has one job and no knowledge of the layer above it.
3. **Dependency injection at the boundary.** Services receive dependencies (repo, config, logger) as arguments or via container — never using global imports.
4. **Config from environment.** No secrets or environment-specific values in source code. All externalized to environment variables parsed at startup.
5. **One source of truth for error types.** Define a small set of domain errors (NotFound, Unauthorized, ValidationError) and map them to HTTP codes only in the route/controller layer.

## Step-by-step guide
1. **Create the top-level folders:**
   ```
   src/
     config/        ← env parsing, validated at startup
     routes/        ← routing declarations (paths + method bindings)
     controllers/   ← request/response handling, no business logic
     services/      ← business logic, framework-agnostic
     repositories/  ← all database access
     models/        ← data models and types/schemas
     middleware/    ← auth, logging, error handling
     utils/         ← pure helper functions
   ```
2. **Parse and validate config at startup.** Use `zod` (TS) or Pydantic (Python) to validate all env vars before the server starts. Crash early on misconfiguration.
3. **Route files are routers only.** No logic in route files — only path + method + middleware + controller reference.
4. **Controllers handle HTTP.** Parse request body/params → call service → map result to HTTP response. Catch domain errors and map to status codes.
5. **Services own business logic.** No `req`, `res`, or database driver imports. Receive repositories and config via arguments.
6. **Repositories own queries.** Only the repository file imports the ORM/driver. Services call repository methods like `userRepo.findByEmail(email)`.
7. **Define domain errors.** Create an `errors.ts` (or `errors.py`) with classes like `NotFoundError`, `ConflictError`. Controllers catch these and return the correct HTTP code.

## The right way vs the wrong way

| Wrong | Right |
|-------|-------|
| Business logic inside a route handler | Route → Controller → Service separation |
| ORM query inside a controller | All queries in repository layer only |
| `process.env.SECRET` inside a service | Inject config object from startup via DI |
| Generic `500` for all errors | Typed domain errors mapped to specific HTTP codes in controller |
| `console.log` for logging | Structured logger (pino, loguru, zap) injected as a dependency |
| Circular imports between layers | Strict dependency direction: routes → controllers → services → repos |

## Stack-specific notes
**Node.js (Express/Fastify/Hono):** Use `zod` for env validation (`dotenv` + `zod.parse(process.env)`). Fastify and Hono have built-in DI patterns; Express uses manual injection.
**Python (FastAPI/Django):** FastAPI's dependency injection system (`Depends()`) is the idiomatic way to inject repos and services. Pydantic settings (`BaseSettings`) replaces manual env parsing.
**Go:** Structure by `cmd/`, `internal/`, `pkg/`. Business logic in `internal/service`, HTTP handlers in `internal/handler`. Use `wire` or manual struct injection for DI. Never use global state.

## Common mistakes
1. **"God route file" with all logic inline.** Difficult to test, review, and onboard. Split into controller + service immediately.
2. **Config imported directly inside a service.** Makes the service impossible to unit test in isolation. Pass config as a constructor argument.
3. **No startup validation of environment.** Runtime panic hours after deploy when a missing env var is first accessed. Always validate the full config at startup.
4. **Mixing HTTP concerns into services.** A service that returns `{ status: 404, body: ... }` is coupled to HTTP. Services return domain values or throw domain errors only.

## Checklist
- [ ] Config parsed and validated at startup using schema validation (zod, Pydantic, etc.)
- [ ] Routes contain only path declarations and controller bindings
- [ ] Controllers contain no business logic — only HTTP parse/respond
- [ ] All business logic lives in service layer
- [ ] All database queries isolated in repository layer
- [ ] Domain error types defined and mapped to HTTP codes in controller layer
- [ ] Structured logger injected as dependency (not global import)
- [ ] No framework-specific imports inside service or repository files
- [ ] Services are unit-testable without starting an HTTP server
- [ ] Folder structure documented in README for new developers
