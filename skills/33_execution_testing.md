# SKILL 3.4 -- How to Write Tests That Actually Catch Bugs

## What this skill is

This skill teaches how to write tests that find real bugs before users do -- not
tests that just verify code passes itself. The majority of tests written in the
industry are written to the happy path and test mostly that the code was typed
correctly. This skill teaches how to think like a bug, design tests that cover
failure modes, and build a test suite that gives genuine confidence before deployment.

---

## When to use this skill

- When writing tests for any new feature
- When adding tests to an existing untested codebase
- When setting up a new test infrastructure
- When a test suite exists but is not catching regressions
- When designing an E2E test for a critical user journey

---

## Full Guide

### The test pyramid

```
                    /\
                   /E2E\         Few (5-10 per major user journey)
                  /------\
                 /  Integ  \     Moderate (1-3 per API endpoint)
                /------------\
               /  Unit Tests  \  Many (1 per function, multiple scenarios each)
              /----------------\
```

**Unit tests:** Test a single function or method in isolation. Dependencies are mocked.
Fast. Run on every save. Should be hundreds in a large codebase.

**Integration tests:** Test a complete request-response cycle against a real (test) database.
No mocks for DB. Slower. Run before every commit.

**E2E tests:** Test a complete user journey through the actual UI.
Playwright or Cypress. Slowest. Run before deployment.

**Wrong pyramid (anti-pattern):**
- Many E2E, few unit tests  → Suite is slow and brittle (UI changes break everything)
- All unit, no integration  → Misses bugs that only appear at layer boundaries
- Only happy path tests     → False confidence, no bug-catching ability

---

### Step 1: Test naming convention

Every test name answers: GIVEN what state, WHEN what action, THEN what outcome.

**Pattern:** `[unit] when [condition], [should/returns/throws] [expected result]`

Examples:
```
UserService.updateProfile
  when user does not exist → throws NotFoundError
  when email is already taken by another user → throws ConflictError
  when input is valid → updates the user record and returns the updated profile
  when user is soft-deleted → throws NotFoundError

AuthService.login
  when password is incorrect → throws UnauthorizedError
  when account is locked → throws ForbiddenError with account_locked code
  when email is not verified → throws ForbiddenError with email_not_verified code
  when credentials are valid → returns access token and sets refresh token cookie
```

---

### Step 2: Arrange / Act / Assert structure

Every test body has three sections, strictly separated:

```typescript
test('UserService.updateProfile when user does not exist → throws NotFoundError', async () => {
  // ARRANGE: set up the preconditions
  const userId = 'non-existent-uuid'
  const input = { first_name: 'Alice' }
  userRepository.findById.mockResolvedValue(null)

  // ACT: perform the operation under test
  const act = () => userService.updateProfile(userId, input)

  // ASSERT: verify the outcome
  await expect(act()).rejects.toThrow(NotFoundError)
  await expect(act()).rejects.toMatchObject({ code: 'NOT_FOUND' })
})
```

Rules:
- Never merge the sections. Arrange is pure setup. Act is one call. Assert is verification.
- Test one thing per test. If you have multiple assertions about different behaviors, split into multiple tests.
- Never put assertions in the Arrange section (no `expect` there).

---

### Step 3: Unit testing a service function

For a service function, mock all dependencies and test only the logic:

```typescript
// What to mock:
// - Repository methods (findById, create, update, delete)
// - External service clients (email sender, Stripe, etc.)
// - Utilities that are tested elsewhere (token generator, etc.)

// What NOT to mock:
// - The function under test itself
// - Small utility functions that have no side effects
// - Pure data transformations
```

Scenarios to always test for any service method:
1. Happy path (all valid input, all dependencies return expected results)
2. Resource not found (when a required entity doesn't exist)
3. Permission denied (when caller lacks access)
4. Duplicate/conflict (when an action would create a conflict)
5. External service failure (when a dependency throws)
6. Boundary values (minimum valid input, maximum valid input, empty input)

---

### Step 4: Integration testing an API route

An integration test makes a real HTTP request to a real route handler against
a real test database. No mocks except external services (email, Stripe).

Setup pattern:
```typescript
// Before suite: spin up test app + migrate test DB
// Before each test: start transaction (so tests are isolated)
// After each test: rollback transaction (DB returns to clean state)
// After suite: close DB connection and server
```

Scenarios to always test for any API endpoint:
1. Unauthenticated request → 401
2. Authenticated but lacks permission → 403
3. Valid request → expected 2xx with correct body shape
4. Invalid body (missing required field) → 400 VALIDATION_ERROR with field details
5. Resource conflict → 409 CONFLICT
6. Resource not found → 404 NOT_FOUND
7. Rate limit exceeded → 429 RATE_LIMIT_EXCEEDED

---

### Step 5: The 5 categories of failure to always test

Beyond happy path, test these 5 failure categories:

1. **Invalid input** -- missing fields, wrong types, out-of-range values, malformed data
2. **Missing resource** -- the thing being operated on does not exist
3. **Permission violation** -- the actor does not have permission to perform the action
4. **State conflict** -- the action is valid but conflicts with current system state
5. **External dependency failure** -- the database is down, Stripe returns an error, email fails

If any of these 5 categories is not tested for a feature, the test suite is incomplete.

---

### Step 6: Testing authentication and authorization

Authentication tests (do these for the auth system, not every route):
- Valid credentials → returns token
- Wrong password → returns 401
- Non-existent account → returns 401 (same message as wrong password -- enumeration prevention)
- Expired access token → returns 401 with TOKEN_EXPIRED code
- Expired refresh token → returns 401 with REFRESH_TOKEN_EXPIRED code
- Reused refresh token (theft simulation) → returns 401 with TOKEN_REUSE_DETECTED, all sessions revoked

Authorization tests (do these for every protected route):
```typescript
test('PATCH /users/:id returns 403 when authenticated user tries to update different user', async () => {
  const alice = await createTestUser()
  const bob = await createTestUser()
  const token = await getAuthToken(alice)

  const res = await request(app)
    .patch(`/api/v1/users/${bob.id}`)
    .set('Authorization', `Bearer ${token}`)
    .send({ first_name: 'Hacker' })

  expect(res.status).toBe(403)
  expect(res.body.error.code).toBe('FORBIDDEN')
})
```

---

### Step 7: Test data setup (factories and builders)

Never hard-code test data inline. Use factories:

```typescript
// Factory pattern for test data
async function createTestUser(overrides = {}) {
  return db.user.create({
    data: {
      id: crypto.randomUUID(),
      email: `test-${Date.now()}@example.com`,
      first_name: 'Test',
      last_name: 'User',
      hashed_password: await bcrypt.hash('ValidPass123!', 12),
      is_verified: true,
      ...overrides
    }
  })
}

// Usage:
const adminUser = await createTestUser({ role: 'admin' })
const unverifiedUser = await createTestUser({ is_verified: false })
```

Rules:
- Test data is always created in the DB for integration tests, not mocked
- Factories have sensible defaults and accept overrides
- Random or timestamped values prevent accidental test contamination

---

### Step 8: E2E test coverage

Cover these critical paths with E2E tests (not everything -- only what matters most):

| User Journey | Priority |
|---|---|
| Signup → verify email → login | Critical |
| Password reset complete flow | Critical |
| Subscription purchase flow | Critical |
| Core feature happy path | High |
| Logout and re-login | High |
| Error state visibility (submitting bad form) | Medium |

E2E tests use test accounts and Stripe test mode. They must not affect production data.

---

### Common test anti-patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| Testing implementation details | Test breaks when you refactor without changing behavior | Test behavior, not implementation |
| Asserting on mocks being called | Tests pass even when logic is wrong | Assert on the output of the function |
| One test that tests everything | Hard to know what broke | One behavior per test |
| Sharing mutable state between tests | Tests affect each other (flaky) | Reset state before each test |
| Mocking the function under test | You are not testing anything real | Never mock what you are testing |
| Tests that always pass | Zero bug-catching value | Use TDD or write test first, watch it fail, then fix |
| Happy path only | Massive false confidence | Always test the 5 failure categories |
| No assertion on error messages | Tests pass with wrong error | Assert on code AND message shape |

---

### Test quality measurement

Coverage metrics to enforce:
```
Line coverage:    80% minimum
Branch coverage:  70% minimum (every `if` has both branches tested)
Function coverage: 90% minimum
```

Coverage does NOT guarantee quality. A 100% coverage test suite that only tests
the happy path is 0% effective at finding bugs. Coverage is a floor, not a ceiling.

---

## What to avoid

DO NOT write tests only for the happy path.

DO NOT share mutable state between tests.

DO NOT mock the function or method you are testing.

DO NOT test implementation details (that a specific internal function was called).

DO NOT skip integration tests and rely only on unit tests.

DO NOT write tests AFTER the feature is complete as an afterthought.

DO NOT leave empty catch blocks in tests that swallow assertion failures.

---

## Checklist

Before marking any feature as "tested":

- [ ] Unit tests written for all service methods
- [ ] All 5 failure categories tested (invalid input, missing resource, permission, conflict, external failure)
- [ ] Integration tests written for every API endpoint
- [ ] Auth tests: unauthenticated → 401, unauthorized → 403
- [ ] Each test has clear Arrange / Act / Assert sections
- [ ] Test names follow when/should/throws convention
- [ ] Test data created via factories, not hard-coded inline
- [ ] DB state resets between tests (transaction rollback or truncation)
- [ ] No `any` types in test files
- [ ] E2E test exists for the critical user journey this feature enables
- [ ] Coverage thresholds met (80% lines, 70% branches)
- [ ] No flaky tests (run the suite 3 times, all passes)
- [ ] All tests pass in CI, not just locally
