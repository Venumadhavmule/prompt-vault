# SKILL 2.1 -- How to Plan a New Feature End to End

## What this skill is

This skill is the complete planning process for any new feature -- from raw
idea to an execution-ready specification that a developer or AI agent can build
from without ambiguity. A feature built without this plan will be built wrong,
tested inadequately, or discovered to be missing critical pieces mid-implementation.
This skill eliminates all of that by front-loading the thinking.

---

## When to use this skill

- Before implementing any new feature
- Before picking up a ticket from a backlog
- When a non-technical stakeholder describes a need and you must turn it into a buildable spec
- When an AI agent asks "what do you want me to build?" and you need to provide a complete brief

---

## Full Guide

### Step 1: Write the User Story

A user story forces you to name who benefits, what they do, and why.
Without the "why," you cannot evaluate whether your implementation actually serves the user.

**Format:**
```
As a [type of user],
I want to [action/capability],
so that [outcome/benefit].
```

**Good examples:**
```
As a registered user,
I want to reset my password via email,
so that I can regain access when I forget my credentials.

As a billing admin,
I want to download a PDF invoice for any past payment,
so that I can submit expenses to my finance department.
```

**Red flags in a user story:**
- The user is "the system" (e.g., "as the system, I want to send emails") -- this is a
  technical task masquerading as a user story. Reframe it from the real user's perspective.
- No "so that" clause -- means you do not know why this matters. Do not build it without knowing.
- Multiple unrelated capabilities in one story -- split into multiple stories.

---

### Step 2: Write the Acceptance Criteria

Acceptance criteria are the contract. They are the specific, observable, testable
conditions that determine whether the feature is done. Every condition must be binary:
it either passes or it does not.

**Format:** Bullet list of "Given / When / Then" or plain "The system must..." statements.

**Examples for password reset:**
```
ACCEPTANCE CRITERIA:

- User can submit their email address at /forgot-password
- If the email exists in the system, a password reset email is sent within 30 seconds
- If the email does NOT exist, the response is the same (to prevent email enumeration)
- The email contains a unique link valid for exactly 60 minutes
- After 60 minutes, the link returns an "expired" error
- After the link is used once successfully, subsequent uses return "already used" error
- Successful reset invalidates all existing refresh tokens for that user (forces re-login)
- Rate limit: maximum 3 reset requests per email per hour
- Invalid emails (malformed) are rejected by the form before submission
```

**What makes bad acceptance criteria:**
- "The password reset should work" -- not testable
- "Looks good on mobile" -- not specific enough (which screen sizes? which browsers?)
- "Fast response" -- not measurable (define: under 200ms? under 1 second?)

---

### Step 3: Define the Data Model

Identify everything this feature needs from the database:

**Questions to answer:**
1. Does this feature require new tables? What are they?
2. Does this feature require new columns on existing tables?
3. What are the relationships (foreign keys)?
4. What constraints are required (unique, not null, check)?
5. What indexes are needed (foreign keys, search columns, composite)?
6. What is the migration path? Can it be done live without downtime?
7. Does existing data need to be backfilled?

**For the password reset example:**
```sql
-- New table needed:
password_reset_tokens (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  token_hash  TEXT NOT NULL,        -- store HASH not raw token
  expires_at  TIMESTAMPTZ NOT NULL,
  used_at     TIMESTAMPTZ,          -- null = not used, set when consumed
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
)

-- Indexes:
CREATE INDEX ON password_reset_tokens(token_hash);     -- lookup by token
CREATE INDEX ON password_reset_tokens(user_id);        -- cleanup by user
CREATE INDEX ON password_reset_tokens(expires_at);     -- cleanup of expired tokens
```

---

### Step 4: Define the API Contract

For every endpoint this feature requires, write the complete contract BEFORE
writing any implementation:

```
METHOD   PATH            AUTH REQUIRED   RATE LIMIT
POST     /auth/forgot-password   No      3 per email per hour
POST     /auth/reset-password    No      5 per IP per hour

---
POST /auth/forgot-password

Request body:
  { "email": "string (email format, required)" }

Success response (200):
  { "message": "If that email is registered, a reset link has been sent." }
  NOTE: Always return 200 regardless of whether email exists (prevents enumeration)

Error responses:
  400 - { "error": { "code": "VALIDATION_ERROR", "message": "Invalid email format" } }
  429 - { "error": { "code": "RATE_LIMIT_EXCEEDED", "message": "Too many attempts. Try again in 60 minutes." } }

---
POST /auth/reset-password

Request body:
  {
    "token": "string (required)",
    "password": "string (min 8 chars, required)"
  }

Success response (200):
  { "message": "Password updated. Please log in with your new password." }

Error responses:
  400 - VALIDATION_ERROR (password too short, fields missing)
  400 - INVALID_TOKEN (token not found or already used)
  400 - TOKEN_EXPIRED (token past expiry)
  429 - RATE_LIMIT_EXCEEDED
```

Every API contract must answer: What goes in, what comes out, and what can fail?

---

### Step 5: Define the UI/UX Flow

Map every screen, state, and transition the user experiences.

**States to always define:**
- Loading (what does the user see while waiting?)
- Empty (what do they see with no data?)
- Error (what do they see when something fails?)
- Success (what do they see when it works?)
- Disabled (are any parts of the UI conditionally disabled?)

**For password reset:**
```
Screen 1: /forgot-password
  - States: idle | loading | success | error
  - Idle: email input field, submit button
  - Loading: button disabled, spinner visible
  - Success: "Check your email" message, no form
  - Error: inline error message, form still editable

Screen 2: /reset-password?token=[token]
  - States: loading (validating token) | valid | expired | already_used | success
  - Loading: spinner while token is verified
  - Valid: password input + confirm password + submit button
  - Expired: "This link has expired. Request a new one." + link to /forgot-password
  - Already used: "This link has already been used. Request a new one."
  - Success: "Password updated!" + redirect to /login after 2 seconds
```

---

### Step 6: Write the Test Plan

Define what must be tested at each level BEFORE writing tests:

**Unit tests (service layer):**
- Happy path for every service method
- Every explicit error condition (expired token, already used, rate limited)
- Edge cases (empty input, malformed input, boundary values)

**Integration tests (API routes):**
- Full request flows against a real (test) database
- Auth-protected routes with valid and invalid tokens
- Rate limiting validation (sends more than limit, expects 429)

**E2E tests (user journey):**
- Complete reset flow: request → email → link → new password → login
- Expired token flow
- Rate limit flow

**What NOT to test:**
- Stripe's email delivery (test that you called the service with correct params, not that Stripe worked)
- Database internals (test through the repository interface, not raw SQL)

---

### Step 7: Identify Dependencies

Before starting, identify everything this feature depends on that you do NOT control:

| Dependency | Type | Impact |
|---|---|---|
| Email service (Resend/Postmark) | External API | Feature cannot work without it. Mock in tests. |
| Redis (for rate limiting) | Infrastructure | Required in all envs where this runs. |
| Existing auth system | Internal | Must not break existing login/logout flows. |
| ENV vars: RESET_TOKEN_EXPIRY_MINUTES | Config | Must be added to .env.example |

Also check:
- Does this feature need a feature flag for gradual rollout?
- Does this feature require a third-party service account or API key?
- Is any part of this blocked waiting for another team?

---

### Step 8: Estimate and Communicate

After the complete plan is written, assign sizes to each build subtask (from Skill 1.2)
and communicate the estimate.

**Estimate format:**
```
FEATURE: Password Reset Flow
ESTIMATED EFFORT:
  DB migration + rollback:     S (30 min)
  PasswordResetService:        M (1.5 hr)
  Email template:              S (45 min)
  POST /forgot-password route: S (30 min)
  POST /reset-password route:  S (30 min)
  Rate limiting middleware:    S (30 min)
  Unit tests:                  M (1.5 hr)
  Integration tests:           M (1.5 hr)
  Frontend screen 1:           M (1 hr)
  Frontend screen 2:           M (1 hr)
  .env.example + docs:         S (15 min)

TOTAL ESTIMATE: ~9.5 hours
DEPENDENCIES: Email service must be configured, Redis must be running
```

---

### The Feature Planning Document (Full Template)

```markdown
# Feature: [Name]

## User Story
As a [user type], I want to [action], so that [outcome].

## Acceptance Criteria
- [ ] [Observable condition 1]
- [ ] [Observable condition 2]
- [ ] [Observable condition N]

## Data Model Changes
### New Tables
[SQL DDL or Prisma model]

### Modified Tables
[Column additions with migration notes]

### Indexes Required
[index strategy]

## API Contract
### Endpoints
[Full contract for each endpoint]

## UI/UX Flow
### Screens
[Screen-by-screen state breakdown]

## Test Plan
### Unit Tests
[List]
### Integration Tests
[List]
### E2E Tests
[List]

## Dependencies
[Table of external and internal dependencies]

## Estimates
[Task breakdown with sizes]

## Definition of Done
- [ ] All acceptance criteria pass
- [ ] All planned tests written and passing
- [ ] No TypeScript errors
- [ ] PR reviewed and merged
- [ ] Feature works in staging
- [ ] .env.example updated
- [ ] OpenAPI spec updated
```

---

## What to avoid

DO NOT start implementation before the acceptance criteria are written.

DO NOT write vague acceptance criteria. Every criterion must be binary (pass/fail).

DO NOT define the API contract only as "we will figure it out during implementation."

DO NOT skip the UI state breakdown. Loading, empty, and error states are features, not afterthoughts.

DO NOT leave dependencies unidentified. Missing a required service or env var causes mid-sprint blockers.

DO NOT estimate before the plan is complete. An estimate without a plan is a guess.

---

## Checklist

Before marking a feature as "planned and ready to build":

- [ ] User story is written with a clear "so that" clause
- [ ] Acceptance criteria are binary and testable
- [ ] Data model changes are fully specified with column types and constraints
- [ ] Indexes are planned for all foreign keys and search columns
- [ ] Forward AND rollback migration are planned
- [ ] Full API contract defined (every endpoint: method, path, body, response, error states)
- [ ] All UI states defined (loading, empty, error, success) for every screen
- [ ] Test plan covers unit, integration, and E2E
- [ ] All external and internal dependencies identified
- [ ] New environment variables listed and added to .env.example plan
- [ ] Feature estimate communicated
- [ ] A developer or AI agent could start building from this document with zero clarifying questions
