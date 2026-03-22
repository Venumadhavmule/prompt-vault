# SKILL 01-C — How to Identify What You Do Not Know and What to Assume
Category: Foundations and Mindset
Applies to: All stacks, all task types, all AI coding agents

## What this skill covers
The most dangerous engineering state is confident ignorance — moving forward on an assumption you believe to be true but have not verified. This skill covers how to systematically surface every unknown before starting work, classify each unknown as "must verify" or "safe to assume", and handle the ones that cannot be resolved by using explicit, marked assumptions rather than silent guesses.

## When to activate this skill
- When working in an unfamiliar codebase or domain
- When a task description leaves implementation details unspecified
- When you are about to make a choice that will be hard to reverse
- When an AI agent is generating code without access to all referenced files
- Before making any architectural or data model decision

## Core principles
1. **Named assumptions are safe; unnamed assumptions are time bombs.** An assumption you write down can be reviewed. An unconscious one cannot.
2. **Distinguish "must know" from "safe to assume".** Some unknowns block progress. Others can proceed with a reasonable default.
3. **Verify by reading, not by guessing.** Import paths, function signatures, existing patterns — these must be read from the codebase, not invented.
4. **Mark every assumption in the code.** If you write code that relies on something unverified, mark it with a comment indicating the assumption.
5. **The cost of a wrong assumption increases with time.** Surface unknowns at the start, not at code review.

## Step-by-step guide
1. After reading the task (SKILL 01-A), write two columns: KNOWNS and UNKNOWNS.
2. For every unknown, classify it: **Blocking** (cannot proceed without knowing) or **Non-blocking** (can proceed with a reasonable assumption).
3. For blocking unknowns: find the answer by reading the codebase, schema, or documentation before writing any code.
4. For non-blocking unknowns: write the assumption explicitly in one sentence: "Assuming X because Y" and mark it in any generated code.
5. Check the following sources to resolve unknowns before assuming:
   - Existing similar features in the codebase (read them, copy their patterns)
   - Schema files for column names, types, relationships
   - Existing test files for expected behavior patterns
   - Environment variable files (.env.example) for available config
   - Existing error format in the API for response shape
6. For any unknown that cannot be resolved by reading: write it as an explicit PLACEHOLDER or TODO comment.
7. Never generate a database column name, function name, or import path that was not verified in a source file.
8. After resolving all blocking unknowns and documenting all assumptions, proceed with implementation.

## The right way vs the wrong way

| Wrong | Right |
|-------|-------|
| Use `user.id` without checking if the field is `id` or `userId` in schema | Read `prisma/schema.prisma` and confirm the exact field name |
| Invent a utility function path like `@/lib/auth` without checking it exists | Read the directory or an existing import to confirm the path |
| Assume JWT is the auth method without checking | Read an existing authenticated route to see what middleware is used |
| Write `import { db } from "@/lib/db"` without confirming that file exists | Check that `src/lib/db.ts` or equivalent exists before using it |
| Say "I'll figure out the auth pattern later" | Resolve the auth mechanism before writing any protected route |
| Proceed with a wrong assumption silently | Mark the assumption: `// ASSUMES: User.id is a string cuid (from schema)` |

## Stack-specific notes
**Node.js / TypeScript:** Use TypeScript's own compiler as a verification tool — `tsc --noEmit` will surface type shape mismatches fast. Read `tsconfig.json` to understand path aliases before using them.
**Python:** Read `pyproject.toml` or `requirements.txt` to verify dependencies exist before importing them. Check `models.py` or equivalent for actual column names.
**Go:** Read the `go.mod` file to confirm third-party packages exist. Go's strict compilation will surface missing package errors, but dependency mismatches must be caught before compile.
**All stacks:** The unknowns classification process is identical. What differs is the canonical source files to read per stack.

## Common mistakes
1. **Inventing import paths.** The most common AI agent failure — writing `import { x } from "@/utils/helpers"` when that file does not exist.
2. **Assuming database column names.** Column names are the single biggest source of silent runtime failures. Always read the schema.
3. **Assuming the error response format.** If the existing API returns `{error: {code, message}}` and new code returns `{error: string}`, the client breaks silently.
4. **Not marking assumptions in code.** Assumptions left unmarked become invisible technical debt.
5. **Asking multiple clarifying questions at once.** Forces the human to write an essay when they expected a simple answer. Ask the most blocking question only.

## Checklist
- [ ] Known vs unknown items listed explicitly before starting
- [ ] Every unknown classified as blocking or non-blocking
- [ ] All blocking unknowns resolved by reading source files
- [ ] Schema/database column names verified against actual schema file
- [ ] Import paths verified against actual directory structure
- [ ] Existing error response format read and referenced
- [ ] Existing auth mechanism read from an existing route
- [ ] Every non-blocking assumption written in one sentence
- [ ] No invented utility functions, library names, or import paths
- [ ] PLACEHOLDER comments placed for anything unresolvable
- [ ] Only one clarifying question sent if human input is required
