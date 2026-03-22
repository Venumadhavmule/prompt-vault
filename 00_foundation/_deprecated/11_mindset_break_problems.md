# SKILL 1.2 -- How to Break Any Problem Into Subtasks

## What this skill is

This skill teaches the systematic decomposition of any engineering problem into
executable, ordered, right-sized subtasks. The inability to decompose is the root
cause of being "stuck", writing sprawling uncommittable PRs, missing hidden work,
and failing to communicate progress. This skill eliminates all of those.

It applies equally to human developers and AI coding agents. An AI agent that
cannot decompose a task will either produce incomplete output or collapse complexity
into a single monolithic action that is wrong in unpredictable ways.

---

## When to use this skill

- When a task takes more than an estimated 2 hours to complete
- When a task touches multiple layers (DB, API, UI, tests, config)
- When you are about to start working on something that "feels big"
- When planning a feature or a sprint
- When an AI agent is given a complex multi-step instruction
- When mid-execution you realize the task is larger than initially scoped

---

## Full Guide

### Step 1: Define the dependency tree

Before writing any subtask, draw the dependency tree. A dependency tree answers:
"What must exist before I can build this?"

Rules:
- Nodes are units of work
- Edges point from "must exist first" to "can only be built after"
- The things with no incoming edges are your starting points

Example: "Add user profile edit feature"
```
[Types / Interfaces]
        |
[DB Migration (add columns)]
        |
[Repository layer (update user)]
        |
[Service layer (validate + call repo)]
        |
[API Route (PATCH /users/:id)]        [Auth middleware already exists]
        |                                      |
[UI Form (ProfileForm component)]  -------+
        |
[UI Page (ProfilePage)]
        |
[Integration tests] + [E2E tests]
```

Always start with the leaves (bottom-most items with no dependencies).
Never build a node whose dependency is not yet complete.

---

### Step 2: Size every subtask

Use this classification for every subtask:

| Size | Time estimate | Rule |
|---|---|---|
| Small | 15-30 minutes | A single, clearly bounded change. One file or two closely related changes. |
| Medium | 1-2 hours | A coherent group of changes in one layer. Service + its tests. |
| Large | More than 2 hours | MUST be split further. A "large" subtask is not a subtask, it is a feature. |

When in doubt, split. The overhead of splitting is always smaller than the risk of
a monolithic chunk that is hard to review, test, or roll back.

---

### Step 3: The one-sentence rule

If you CANNOT describe a subtask in one sentence, it is too large. Split it.

VALID subtasks:
- "Write the Zod schema for the user profile update request body"
- "Add first_name, last_name, bio, avatar_url columns to the users table migration"
- "Implement UserRepository.updateProfile(userId, data) with Prisma"
- "Add PATCH /api/users/:id route with auth guard and delegating to UserService"
- "Write unit tests for UserService.updateProfile covering success, not found, and validation failure"

INVALID subtasks:
- "Build the profile feature"
- "Do the backend for profile editing"
- "Add profile editing everywhere"

---

### Step 4: The correct ordering for any software task

This is the mandatory ordering. Never deviate from it:

```
1. Types and Interfaces       (define the shape of data first)
2. Zod/Validation Schemas     (define what valid input looks like)
3. Database Schema/Migration  (schema changes before anything uses them)
4. Repository / Data Layer    (database access before business logic)
5. Service Layer              (business logic before HTTP layer)
6. API Layer / Controller     (HTTP concerns after logic is solid)
7. UI Components              (after API contract is confirmed)
8. UI Pages / Screens         (after components exist)
9. Tests (alongside each layer, not at the end)
10. Documentation             (JSDoc, OpenAPI, README updates)
```

WHY this order matters:
- If you build the UI before the API exists, you will reshape the UI to match whatever
  API you ended up building, wasting work
- If you build the service before the DB schema, you will write service logic that
  does not match your actual data model
- Tests written alongside development catch errors immediately. Tests written at the
  end are often written to pass, not to catch bugs

---

### Step 5: Spot hidden subtasks

These subtasks are almost always forgotten. Check for each:

| Hidden subtask | When it applies |
|---|---|
| Database migration | Any time a DB schema changes |
| Rollback migration | Every forward migration needs a rollback |
| Environment variables | Any new third-party service, secret, or config value |
| Error classes / types | Any new category of failure in the system |
| Type definitions | Any new data shape crossing a boundary |
| Zod schema | Any new API input or function input that comes from external sources |
| Seed data update | If new required data must exist in dev/staging |
| .env.example update | Every new env var must be documented |
| OpenAPI spec update | Every new or changed API endpoint |
| CHANGELOG update | Every releasable change |
| Dependency installation | Any new npm package (also check license) |
| Prisma client regeneration | Any Prisma schema change |
| DB index | Any new column used in a WHERE or ORDER BY clause |
| Feature flag | Any feature that should be rolled out gradually |
| Permission/RBAC entry | Any new action or resource needing access control |

Before finalizing your subtask list, go through this list and add any that apply.

---

### Step 6: Communicate the breakdown before executing

Before starting any task of 3+ subtasks, output your breakdown.

Use this format:
```
TASK: [task name]

SUBTASKS IN EXECUTION ORDER:
[ ] 1. [subtask] — [size: S/M] — [layer: types/schema/repo/service/api/ui/test]
[ ] 2. [subtask] — [size: S/M] — [layer: ...]
[ ] 3. [subtask] — [size: S/M] — [layer: ...]
...

HIDDEN SUBTASKS IDENTIFIED:
[ ] - [subtask from the hidden subtask list]

ASSUMPTIONS:
- [any assumption made during decomposition]

ESTIMATED TOTAL: [sum of time estimates]
```

This output serves two purposes:
1. It surfaces mistakes before execution starts (someone else can catch a wrong assumption)
2. It creates a clear progress tracker during execution

---

### Step 7: Handle unexpected complexity mid-execution

When you are executing a subtask and discover it is larger than expected:

1. STOP. Do not push through. Continuing without acknowledgment makes things worse.
2. Assess: is this a SCOPE CHANGE (the task was always larger) or a DISCOVERY (new info)?
3. Re-decompose the affected subtask into smaller pieces
4. Update your subtask list
5. If the total scope has doubled, communicate this before continuing

Do NOT silently absorb additional complexity without communicating it.
The cost of a scope discussion is always smaller than the cost of a wrong implementation discovered at PR review.

---

### Step 8: The subtask breakdown format (template)

```markdown
## Task: [Task Title]

### Type: feature | bug | refactor | infra | review

### Goal
[One sentence restating why this task exists]

### Dependency Tree
```
[Types]
   └── [Migration]
          └── [Repository]
                 └── [Service]
                        └── [API Route]
                               └── [Tests]
```

### Subtask List

| # | Subtask | Size | Layer | Status |
|---|---------|------|-------|--------|
| 1 | Define TypeScript interfaces for [X] | S | Types | [ ] |
| 2 | Write Zod schema for [X] input | S | Validation | [ ] |
| 3 | Create migration: add [columns] to [table] | S | DB | [ ] |
| 4 | Create rollback migration | S | DB | [ ] |
| 5 | Implement [Entity]Repository.[method] | M | Repository | [ ] |
| 6 | Implement [Entity]Service.[method] | M | Service | [ ] |
| 7 | Add [METHOD] /[path] route | S | API | [ ] |
| 8 | Write unit tests for [Service] | M | Test | [ ] |
| 9 | Write integration test for [route] | M | Test | [ ] |
| 10 | Update .env.example if applicable | S | Config | [ ] |
| 11 | Update OpenAPI spec | S | Docs | [ ] |

### Hidden Subtasks Identified
- [ ] [list any hidden subtasks found]

### Assumptions
- [assumption 1]
- [assumption 2]
```

---

## What to avoid

DO NOT start with the UI layer. Always start with types and data.

DO NOT create a subtask you cannot describe in one sentence.

DO NOT leave a subtask sized as "Large." It must be split.

DO NOT skip the dependency tree. Building in the wrong order causes rework.

DO NOT forget hidden subtasks. Missing a migration or .env.example update causes deployment failures.

DO NOT silently absorb scope increases discovered mid-execution.

DO NOT batch multiple logical layers into one subtask (e.g., "Add repo and service and route").

DO NOT write tests last. Tests belong alongside each layer.

---

## Checklist

Before starting any decomposed work, verify:

- [ ] I have drawn the dependency tree
- [ ] Every subtask is described in one sentence
- [ ] No subtask is estimated as "Large" (if so, split it)
- [ ] Subtasks are ordered: types → schema → migration → repo → service → api → ui → tests
- [ ] I have checked the hidden subtask list and added any that apply
- [ ] I have communicated the breakdown before starting execution
- [ ] I have listed all assumptions made during decomposition
- [ ] Tests are listed alongside the layers they cover, not at the end
- [ ] The .env.example is on the list if any new env vars are added
- [ ] Migrations include a rollback entry
- [ ] I know what "done" looks like for each individual subtask
- [ ] I have a plan for communicating if unexpected complexity is discovered
