# SKILL 01-B — How to Break Any Problem Into Ordered Subtasks
Category: Foundations and Mindset
Applies to: All stacks, all task types, all AI coding agents

## What this skill covers
Decomposition is the skill that separates engineers who ship from engineers who spin. Any complex task, when attempted as a single unit, produces partial implementations, forgotten edge cases, and untrackable progress. This skill covers how to take any requirement — from a simple feature to a full system design — and break it into an ordered set of atomic, independently verifiable subtasks that can each be completed, tested, and verified on their own.

## When to activate this skill
- When a task involves more than one layer (database + API + UI)
- When you cannot complete a task in one sitting without losing context
- When a task involves more than three files being changed
- Immediately after completing SKILL 01-A (reading the task)
- When an AI agent needs to work on a multi-step task without losing track

## Core principles
1. **One subtask = one verifiable outcome.** If you cannot confirm a subtask is done without completing the next one, split it further.
2. **Dependencies dictate order.** Types before consumers. Schema before queries. Service before routes. UI before integration tests.
3. **Work inside-out.** Build the innermost layer (data model, types) before the outer layers (API, UI).
4. **Each subtask is independently test-checkable.** When you finish a subtask, you can verify it without needing the whole feature to work.
5. **Write the subtask list before starting any of them.** The act of writing exposes missing steps and wrong ordering.

## Step-by-step guide
1. Write the goal of the feature as one sentence (from SKILL 01-A done-definition).
2. Identify all system layers touched: types, database/schema, migration, repository, service, API handler, UI, tests.
3. List every subtask as: "[Verb] [specific thing]" — e.g. "Add `email_verified` column to User table", not "update schema".
4. Draw the dependency tree: which subtasks cannot start until another is done? Write this as a simple ordered list.
5. Identify which subtasks can be done in parallel (no dependency between them).
6. Order the list: data model → migration → repository/data access → service/business logic → API/interface → UI → tests.
7. Assign a verification method to each subtask: "verify by: running `tsc --noEmit`", "verify by: calling the endpoint with curl".
8. Check the list for missing subtasks: error handling? input validation? auth guard on new route? CHANGELOG entry? documentation?
9. If any subtask still feels too large, split it until every item is completable in under 30 minutes.
10. Do not start step 1 of the implementation until the full subtask list is written.

## The right way vs the wrong way

| Wrong | Right |
|-------|-------|
| "Implement auth feature" as one task | Decompose into: schema, migration, service, routes, middleware, tests |
| Start implementation, discover missing steps midway | Write the full subtask list before touching any code |
| Order: UI first, figure out backend later | Order: types → schema → data layer → service → API → UI → tests |
| A subtask is: "Make the login work" | A subtask is: "Implement `authService.login()` returning access and refresh tokens" |
| No verification method per subtask | Every subtask has a specific verification: command, test name, or manual check |
| Keep the decomposition in your head | Write it down — the act of writing reveals missing steps |

## Stack-specific notes
**Node.js / TypeScript:** The mandatory order for any feature: types → Prisma schema → migration → repository functions → service functions → Express/Hono route handler → Zod validation → tests. Never skip this order.
**Python:** types (Pydantic models) → Alembic migration → SQLAlchemy repository → service layer → FastAPI route → pytest tests.
**Go:** struct definitions → repository interface + implementation → service interface + implementation → handler → wire everything in main/router.
**All stacks:** The decomposition method is identical. Only the layer names change per framework.

## Common mistakes
1. **Treating the framework as the first step.** "Create the Express app" is not a subtask — the framework exists. The first real subtask is defining what the feature needs.
2. **Skipping the migration subtask.** Schema changes require migrations. A subtask list missing a migration step will fail at the database layer.
3. **Defining subtasks that skip layers.** "Write the endpoint that reads from the database" is two subtasks: repository + handler.
4. **Not including error handling as a subtask.** Error paths are not extensions of the happy path — they are separate subtasks.
5. **Not including tests as subtasks.** Tests are not an add-on. Each business logic unit has a corresponding test subtask.
6. **Making the list too long.** If a subtask list has more than 15 items, the scope is too large. Break the feature into two phases.

## Checklist
- [ ] Feature goal written as one sentence before decomposition begins
- [ ] All system layers identified (data, service, API, UI, tests)
- [ ] Every subtask written as "[Verb] [specific thing]"
- [ ] Dependency ordering applied: inner layers before outer layers
- [ ] Each subtask has a specific verification method
- [ ] Error handling subtasks included
- [ ] Input validation subtask included
- [ ] Auth/authorization guard included if needed
- [ ] Tests listed as explicit subtasks, not as a final catch-all
- [ ] No subtask requires more than 30 minutes to complete
- [ ] Full list written before any implementation started
