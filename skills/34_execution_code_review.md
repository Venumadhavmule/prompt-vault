# SKILL 3.5 -- How to Review Code Like a Senior Engineer

## What this skill is

This skill teaches the mental model and practical process for reviewing code in
a way that actually catches problems before they reach production. Most code reviews
are superficial: they check style and obvious mistakes. A senior review applies six
distinct lenses to every PR and produces feedback that is specific, actionable, and
categorized by severity. This skill applies equally to human reviewers and AI agents
reviewing generated code.

---

## When to use this skill

- Before approving any pull request
- When an AI agent is performing a self-review of generated code
- When reviewing your own code before opening a PR
- When conducting a security audit of a codebase

---

## Full Guide

### The reviewer mindset

A good code review is an act of collaboration, not gatekeeping. The goal is:
- Making the codebase better
- Catching problems before they cost 10x as much to fix
- Sharing knowledge across the team

Use this framing:
- Prefer questions over demands: "What's the thinking behind this approach?" rather than "Change this."
- Distinguish your level of conviction: "Must fix" vs "Should fix" vs "Suggestion."
- Acknowledge good work explicitly. "This is a clean approach" is worth saying.

---

### The 6 dimensions of every review

Review every PR through all 6 lenses, in this order:

---

#### Dimension 1: Correctness

Does the code actually do what it says it does?

Checks:
- Read the logic carefully. Does the happy path produce the right output?
- Trace through the failure paths. What happens when X is null? When the DB call fails?
- Are there off-by-one errors?
- Are conditions correct (using `&&` where `||` is needed, etc.)?
- Are race conditions possible? (two requests modifying the same record simultaneously)
- Does the function return the correct value in all branches?
- Are async operations awaited everywhere they need to be?
- Are promises being propagated to the caller, not silently ignored?

Red flags:
- Missing `return` in async functions
- `if (x !== null)` instead of `if (x != null)` when undefined is also possible
- Using `==` instead of `===` in JavaScript/TypeScript
- `.catch(() => {})` that swallows errors
- Async loop operations using `.forEach` instead of `for...of` (forEach does not await)

---

#### Dimension 2: Security

Does this code introduce vulnerabilities?

Apply OWASP Top 10 as a mental checklist:

| Vulnerability | What to look for |
|---|---|
| Injection (SQL/command) | Raw string concatenation into queries. Never `WHERE id = ${userId}`. |
| Broken auth | Is auth middleware applied? Is the token correctly verified? |
| Sensitive data exposure | Are passwords, tokens, or PII returned in responses or logged? |
| Broken access control | Does the code verify the resource belongs to the authenticated user? |
| Security misconfiguration | CORS is `*`? Error messages expose stack traces? Default credentials? |
| SSRF | User-provided URLs being fetched server-side without validation? |
| XSS | User-controlled content rendered in HTML without escaping? |
| Insecure direct object reference | Is `req.params.id` used directly to fetch without ownership check? |

High-severity security findings block the merge. No exceptions.

---

#### Dimension 3: Performance

Will this code cause performance problems at scale?

Checks:
- N+1 query problem: Is a DB query being made inside a loop?
  ```typescript
  // N+1 bug: 1 query for users + N queries for each user's orders
  for (const user of users) {
    user.orders = await db.order.findMany({ where: { userId: user.id } })
  }
  // Fix: use eager loading / include / JOIN
  ```
- Missing pagination on list queries (returning unbounded result sets)
- Missing database indexes for columns in WHERE/ORDER BY clauses
- Synchronous blocking operations on the event loop (CPU-heavy work without async/worker threads)
- Unnecessary re-computation inside loops (move constant work outside the loop)
- Memory leaks (subscriptions, intervals, or streams not cleaned up)
- Large payloads being returned when field selection could reduce them

---

#### Dimension 4: Maintainability

Will the next developer be able to understand and modify this in 6 months?

Checks:
- Functions are small and do one thing (rule of thumb: > 40 lines is a review flag)
- Function and variable names are descriptive (no single-letter variables outside of indexes)
- Magic numbers and strings are extracted to named constants
- Complex logic has a comment explaining WHY (not what -- the code shows what)
- No duplicated logic that could be extracted to a shared utility
- No deep nesting (> 3 levels of indentation is a review flag -- use early returns)
- Abstractions are at the right level (not too generic, not too specific)

---

#### Dimension 5: Test Coverage

Are the tests adequate?

Checks:
- Does a test exist for this change?
- Are the 5 failure categories covered (invalid input, missing resource, permission, conflict, external failure)?
- Are tests testing behavior or implementation details?
- Will these tests catch a regression if the underlying logic breaks?
- Is the test coverage percentage maintained or improved?

If there are no tests for a new feature, this is a "must fix" even if everything else looks good.

---

#### Dimension 6: Documentation / DX

Will other developers and AI agents be able to use this correctly?

Checks:
- Public functions have JSDoc with parameters, return type, and what it does
- New API endpoints are reflected in OpenAPI spec or inline documentation
- New environment variables are added to `.env.example` with description
- Breaking changes are called out in CHANGELOG
- Complex algorithms have comments explaining the approach and its trade-offs
- README is updated if setup or configuration changed

---

### Feedback format rules

Every review comment must be:

**1. Specific:** Point to the exact line. Quote the code if helpful.

**2. Categorized:**
- `[must fix]` -- Blocks merge. Bug, security issue, or missing test.
- `[should fix]` -- Technical debt. Not urgent but should be addressed soon. PR can merge, follow-up ticket required.
- `[suggestion]` -- Optional improvement. No follow-up required.
- `[question]` -- Genuinely curious. Not asking for change, just understanding.
- `[praise]` -- Explicit acknowledgment of good work.

**3. Actionable:** Tell the author what to change, not just what's wrong.

**Bad comment:**
> This is inefficient.

**Good comment:**
> [must fix] This loop makes a DB query on each iteration -- N+1 query. If there are 100 users, this runs 100 queries.
> Fix: use `db.order.findMany({ where: { userId: { in: userIds } } })` before the loop and group results in memory.

---

### How to review a database migration specifically

Migrations get extra scrutiny. They are irreversible in production.

- Is the rollback migration included?
- Is the migration backward compatible? (Is the old code still able to run against the new schema during deploy window?)
- Does adding a NOT NULL column require a default or a backfill?
- Are indexes created with CONCURRENTLY on large tables to avoid locks?
- Is there a check for what happens to existing data?
- Does the migration succeed if run twice (idempotency)?

---

### How to approve a PR with confidence

Only approve when:
- All 6 dimensions reviewed
- All [must fix] issues resolved
- Tests are present and adequate
- You could explain the change to someone else clearly
- You would be comfortable being on-call when this ships

Do not approve out of social pressure, time pressure, or to avoid conflict.
An approval is your endorsement. Own it.

---

## What to avoid

DO NOT leave only positive comments to be nice. Honest feedback is the job.

DO NOT leave only negative comments. Acknowledge good work explicitly.

DO NOT raise style issues as must-fix. Use a linter. Code review is not for style.

DO NOT leave vague comments: "this could be better" -- better how?

DO NOT approve without reading all the changes. Skimming is not reviewing.

DO NOT block on suggestions. Use the category labels to make priority clear.

DO NOT forget to review the tests, not just the implementation.

---

## Checklist

Before submitting a review:

- [ ] All changed files read in full
- [ ] Correctness: logic verified on happy path and all failure paths
- [ ] Security: OWASP Top 10 mental checklist applied
- [ ] N+1 queries checked for any loop with DB access
- [ ] All list endpoints: pagination present
- [ ] Performance: no obvious bottlenecks introduced
- [ ] Maintainability: function sizes, naming, constants, deep nesting checked
- [ ] Test coverage: tests exist, 5 failure categories covered
- [ ] Documentation: JSDoc, .env.example, OpenAPI updated if applicable
- [ ] Migration: rollback exists, backward compatible, indexes CONCURRENTLY if large table
- [ ] All comments are specific, categorized, and actionable
- [ ] All [must fix] issues would be raised regardless of time pressure
- [ ] At least one praise comment if genuine good work was done
- [ ] Ready to be on-call for this change when it ships
