# SKILL 01-E — How to Follow Existing Codebase Conventions
Category: Foundations and Mindset
Applies to: All stacks, all AI coding agents, any task modifying an existing codebase

## What this skill covers
The worst code a developer can write into an existing codebase is technically correct but stylistically foreign. Inconsistency — in naming, structure, error handling, and patterns — makes codebases impossible to maintain. This skill covers how to read and extract the conventions of any codebase before writing a single line, and how to ensure every change is indistinguishable in style from the surrounding code. An AI agent that invents its own conventions in an existing codebase is actively harmful.

## When to activate this skill
- When working in any existing codebase for the first time
- When adding a new file that needs to match the file structure of existing files
- When writing a new function alongside existing functions
- When choosing names for variables, functions, classes, or files
- When deciding how to structure error handling, logging, or validation
- Always — before writing any code in an existing project

## Core principles
1. **Copy existing patterns, do not invent new ones.** For every structural choice, find how it is done already in this codebase and do the same.
2. **Naming is the most visible convention.** Match the exact naming style: camelCase vs snake_case, singular vs plural, `get` vs `fetch` vs `find`.
3. **Consistency beats "better".** A slightly suboptimal pattern applied consistently is better than a superior pattern applied once in isolation.
4. **The test files reveal the expected behavior.** The way existing tests are written reveals what the codebase considers good practice.
5. **Error handling must match.** If existing code throws typed errors and maps them at the handler, do not silently return null — match the pattern.

## Step-by-step guide
1. Before writing any code, read at least two existing files that are similar to what you will create (e.g. if adding a service, read two existing services).
2. Extract conventions from those files:
   - Naming: how are functions named? (`getUserById` or `get_user_by_id`? `fetchUser` or `findUser`?)
   - File structure: what is at the top? Imports, then types, then functions? Class-based or function-based?
   - Error handling: are errors thrown, returned as `Result` types, or returned as objects?
   - Imports: relative imports or path aliases? Barrel files (`index.ts`) or direct imports?
   - Validation: where does input validation happen — at the route level, service level, or both?
3. Check the linter/formatter config (`.eslintrc`, `biome.json`, `prettier.config.js`) to understand enforced rules.
4. Check the git history on similar files to see how the team has evolved the pattern.
5. Match every discovered convention exactly in your new code.
6. If you must deviate from a convention (because it is objectively wrong or the codebase is in transition), add a comment explaining why.
7. Run the linter and formatter before submitting any code (`eslint .` and `prettier --write` or equivalent).

## The right way vs the wrong way

| Wrong | Right |
|-------|-------|
| Introduce `snake_case` in a codebase that uses `camelCase` | Match existing naming: write `getUserById` not `get_user_by_id` |
| Switch from named exports to default exports because you prefer them | Use the export style already present in the surrounding files |
| Invent a new error class when the codebase has an error hierarchy | Extend or use the existing error base class |
| Use `axios` in a file when the rest of the codebase uses `fetch` | Use the existing HTTP client already imported elsewhere |
| Create a new import alias path (`@/helpers`) when `@/utils` exists | Use the existing path alias |
| Write a new file without checking the folder convention for similar files | Read existing files in the same folder to match their structure |

## Stack-specific notes
**Node.js / TypeScript:** Check `tsconfig.json` for path aliases. Check `eslint` and `prettier` configs. Read one existing route, one existing service, and one existing test before writing new ones.
**Python:** Check `ruff.toml`, `pyproject.toml`, or `.flake8` for linting rules. Check existing models for field naming — SQLAlchemy often uses snake_case for columns but Python-side naming may differ.
**Go:** Go is strict by design — `gofmt` and `golint` enforce most style. But still read existing code for package naming, error wrapping patterns, and logging approach.
**All stacks:** The process of reading existing code before writing is universal. What differs is which config files to read for enforced rules.

## Common mistakes
1. **Skipping the "read existing files" step.** This is the entire skill — skipping it means inventing conventions instead of following them.
2. **Reading only one file.** One file may be an outlier. Read at least two to establish the pattern.
3. **Copying the structure but not the naming.** A function with the right structure but wrong name style still violates conventions.
4. **Importing the wrong library.** Multiple HTTP clients, loggers, or validators in one codebase is a convention violation.
5. **Not running the linter.** Linting errors are convention violations enforced by tooling. Submit nothing with lint errors.

## Checklist
- [ ] At least two similar existing files read before writing
- [ ] Naming convention extracted (function names, variable names, file names)
- [ ] Import style identified (path aliases, barrel files, relative imports)
- [ ] Error handling pattern identified and matched
- [ ] Existing validation approach identified and matched
- [ ] Linter/formatter config read
- [ ] Linter passes on new code before submission
- [ ] Formatter run on new code before submission
- [ ] No new libraries introduced when an equivalent already exists in the codebase
- [ ] Any deliberate deviation from convention is commented and justified
