# SKILL 4.3 -- How to Handle Errors and Failures Gracefully

## What this skill is

This skill defines the complete error handling strategy for any application -- from
the error hierarchy design through to logging, user-facing messages, and recovery.
Poor error handling is the fastest way to erode user trust: unhelpful error messages
confuse users, leaked internal errors are security vulnerabilities, and swallowed
errors make debugging impossible. This skill produces applications that fail gracefully,
communicate clearly, and recover automatically where possible.

---

## When to use this skill

- When designing a new service or module
- When implementing a new feature that interacts with external services
- When reviewing code for error handling quality
- When debugging why a system is silently failing
- When setting up the error infrastructure for a new project

---

## Full Guide

### The two types of errors

Every error in an application is one of two types:

**Operational errors** (expected failures)
- Things that can and do happen in normal operation
- The system can handle them gracefully
- Examples: user not found, invalid input, email already exists, payment declined, rate limit exceeded
- How to handle: catch, log at appropriate level, return a structured error response to the client

**Programmer errors** (unexpected bugs)
- Code defects -- something that should never happen but did
- Examples: null reference exception, unexpected undefined, failed assertion, DB schema mismatch
- How to handle: log at ERROR level with full context and stack trace, alert the team,
  return a generic 500 to the client WITHOUT revealing any details

**The key rule:** Never confuse them. Never catch a programmer error and handle it as
if it were operational. Never let an operational error crash the process.

---

### Step 1: Design the error hierarchy

Build a typed error hierarchy from a base class:

```
AppError (base) [isOperational = true]
  ├── ValidationError (400) -- client sent invalid data
  ├── UnauthorizedError (401) -- not authenticated
  ├── ForbiddenError (403) -- authenticated but no permission
  ├── NotFoundError (404) -- resource does not exist
  ├── ConflictError (409) -- action conflicts with current state
  ├── RateLimitError (429) -- too many requests
  ├── UnprocessableError (422) -- input is valid but logically impossible
  └── ServiceUnavailableError (503) -- temporary dependency failure

Domain errors (extend AppError with specific codes):
  ├── AuthError -- token expired, invalid credentials
  ├── PaymentError -- payment declined, invalid card
  ├── SubscriptionError -- plan limit exceeded, subscription required
  └── ExternalServiceError -- upstream API failure
```

**Base AppError fields:**
- `message` -- User-friendly message (may be shown to the client)
- `code` -- All-caps machine-readable code (e.g., `TOKEN_EXPIRED`)
- `statusCode` -- HTTP status code
- `details` -- Optional array of field-level details (for validation errors)
- `isOperational` -- true for expected errors, false/undefined for programmer bugs

---

### Step 2: Write error messages users can act on

A good error message answers: "What happened and what should I do next?"

**Bad error messages:**
- "An error occurred" -- tells the user nothing
- "Something went wrong" -- gives the user no action
- "Error 500" -- meaningless to most users
- "ECONNRESET" -- a raw Node.js error, not a user message

**Good error messages:**
- "That email address is already registered. Try logging in instead, or use forgot password."
- "Your session has expired. Please log in again."
- "The file you uploaded is too large. Maximum size is 5MB."
- "That invitation link has expired. Ask an admin to send a new one."
- "We could not process your payment. Please check your card details and try again."

**Template for actionable error messages:**
"[What happened]. [What the user can do]."

---

### Step 3: Write error messages for developers

The developer-facing error (in server logs) must contain:
- Full stack trace
- Request context (method, path, status, requestId, userId if available)
- The input that caused the error (sanitized -- no passwords or tokens)
- The environment (service name, version, node version)
- Enough context to find the exact line and reproduce the issue

```json
{
  "level": "error",
  "message": "Unhandled error in PATCH /api/v1/users/:id",
  "requestId": "req_abc123",
  "userId": "user_def456",
  "method": "PATCH",
  "path": "/api/v1/users/def456",
  "statusCode": 500,
  "error": {
    "name": "TypeError",
    "message": "Cannot read properties of undefined (reading 'id')",
    "stack": "TypeError: Cannot read properties of undefined...\n    at UserService.updateProfile (/app/services/user.service.ts:45:20)\n    ..."
  },
  "timestamp": "2024-11-15T14:22:33.123Z",
  "service": "api",
  "version": "1.3.2"
}
```

---

### Step 4: Global error handler pattern

The global error handler in the framework (Express/Hono/Fastify) is the single point
where all errors terminate. It determines what the client sees vs what gets logged.

```
For operational AppError:
  → Log at WARN level with context (no stack trace needed for expected errors)
  → Return structured error response: { error: { code, message, details, requestId } }
  → Use the error's statusCode

For unknown errors (not AppError):
  → Log at ERROR level with FULL stack trace and all context
  → Alert via Sentry/PagerDuty
  → Return generic 500 response: { error: { code: "INTERNAL_ERROR", message: "An unexpected error occurred. Reference: [requestId]", requestId } }
  → NEVER include stack trace, DB messages, or file paths in the 500 response
```

---

### Step 5: External service failure handling

When calling Stripe, email services, S3, or any third-party:

**Timeout:** Set an explicit timeout on every HTTP call. Default timeouts are often 30+ seconds.
For user-facing operations, timeout at 3-5 seconds. For background operations, 10-30 seconds.

**Retry with exponential backoff:**
For transient failures (network blip, temporary 503):
```
Attempt 1: immediate
Attempt 2: wait 1 second
Attempt 3: wait 2 seconds
Attempt 4: wait 4 seconds
Max retries: 3-4
Max backoff: 30 seconds
```
Do NOT retry on: 4xx client errors, rate limit errors (use Retry-After header instead)

**Circuit breaker:**
If a service fails 5 consecutive times, open the circuit (stop trying) for 30 seconds.
After 30 seconds, try once (half-open). If it succeeds, close the circuit. If not, keep open.
This prevents cascading failures from hammering a degraded dependency.

**Fallback:**
For non-critical external operations (e.g., sending a notification email):
Catch the error, log it, and continue. The primary operation should not fail because
a notification could not be delivered.

For critical operations (e.g., Stripe payment): do not provide a fallback. Let the
error propagate to the client with a clear "payment unavailable" message.

---

### Step 6: Async errors

Node.js specific: unhandled promise rejections crash the process in Node 15+.

Rules:
- Every `async` function must be awaited or its rejection must be caught
- Use `try/catch` in async route handlers or a global async wrapper
- Set up global handlers for uncaught exceptions:
  ```typescript
  process.on('unhandledRejection', (reason, promise) => {
    logger.error('Unhandled Rejection', { reason, promise })
    // Do NOT exit the process here -- let the global error handler deal with it
  })
  ```

---

### Step 7: Database error handling

| DB Error | Cause | How to handle |
|---|---|---|
| Unique constraint violation | Duplicate record being inserted | Catch, throw ConflictError with user-friendly message |
| Foreign key violation | Referencing non-existent parent record | Catch, throw ValidationError or NotFoundError |
| Connection lost | DB restart or network blip | Retry with backoff. If persistent, throw ServiceUnavailableError |
| Query timeout | Slow query or DB under load | Retry once. If persistent, alert and throw ServiceUnavailableError |
| Deadlock | Concurrent transactions conflicting | Retry with random jitter. Redesign transaction order if frequent. |
| SSL handshake failure | Cert expired or misconfiguration | Alert team immediately -- this needs manual resolution |

**Never** propagate raw Prisma or pg error objects to the client. Always wrap them in domain errors.

---

## What to avoid

DO NOT use `catch (e) {}` (empty catch) anywhere in the codebase.

DO NOT log errors without the requestId and userId -- the context is what makes logs useful.

DO NOT return Prisma error messages, SQL errors, or internal paths to the client.

DO NOT use a single generic "something went wrong" message for all errors -- be specific.

DO NOT retry non-idempotent operations (e.g., charge a credit card twice).

DO NOT let programmer errors (bugs) be silently caught and treated as operational errors.

DO NOT set misleading HTTP status codes (e.g., 200 with `{"success": false}` in the body).

---

## Checklist

For every feature before shipping:

- [ ] Typed error classes from AppError hierarchy used throughout
- [ ] Global error handler in place and distinguishes operational vs programmer errors
- [ ] Operational errors return structured JSON with code, message, and requestId
- [ ] Programmer errors return generic 500 with no internals exposed
- [ ] All async functions have error propagation (no silent swallowing)
- [ ] External service calls have timeouts set
- [ ] External service calls have retry logic for transient failures
- [ ] DB unique constraint violations → ConflictError (not a 500)
- [ ] Error messages tell the user what to do, not just what went wrong
- [ ] Server logs include requestId, userId, method, path, and full stack on errors
- [ ] No passwords, tokens, or sensitive values appear in any error log
- [ ] Sentry or equivalent is configured and captures unhandled errors in production
- [ ] Unhandled promise rejections are captured and alerted on
