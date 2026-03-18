# SKILL 2.3 -- How to Plan an API

## What this skill is

This skill covers the complete process of designing a REST API before writing any
route handler code. An API is a contract between your backend and every client that
will ever call it. Poorly designed APIs are impossible to evolve without breaking
clients. This skill produces an API design that is consistent, secure, evolvable,
and easy for frontend developers to build against without constant back-and-forth.

---

## When to use this skill

- Before writing any new route handler
- At the start of any feature that exposes new endpoints
- When designing a public API that external clients will consume
- When reviewing an API design in a PR
- When a frontend developer asks "what does the API return?"

---

## Full Guide

### Step 1: RESTful resource naming rules

REST APIs model resources (things) not actions (verbs).

**Rules:**
- Use nouns, not verbs: `/users` not `/getUsers`
- Use plural nouns for collections: `/orders` not `/order`
- Nest resources only when a resource is owned by another AND cannot exist independently
- Keep nesting to maximum 2 levels deep

**Correct:**
```
GET    /users              -- list all users
GET    /users/:id          -- get a specific user
POST   /users              -- create a user
PATCH  /users/:id          -- update a user
DELETE /users/:id          -- delete a user

GET    /organizations/:orgId/members        -- members within an org (nested correctly)
POST   /organizations/:orgId/invitations    -- invite to an org
```

**Incorrect:**
```
GET    /getUsers                    -- verb
POST   /createUser                  -- verb
GET    /users/:userId/orders/:orderId/items/:itemId  -- too deeply nested
POST   /processPayment              -- verb
```

**Actions that are not CRUD:** Use POST with an action suffix:
```
POST   /auth/login
POST   /auth/logout
POST   /auth/refresh
POST   /payments/:id/refund
POST   /subscriptions/:id/cancel
POST   /invitations/:id/accept
```

---

### Step 2: Define every endpoint completely

For every endpoint, write the complete contract. Do not leave anything to "figure out later."

**Endpoint contract format:**

```
[METHOD] [PATH]
Auth:        [public | authenticated | role:admin]
Rate limit:  [N requests per M minutes per IP/user]

Request:
  Params:  [path parameters with type and description]
  Query:   [query parameters with type, default, and description]
  Body:    [JSON schema with types, required/optional, constraints]

Success:
  Status:  [HTTP status code]
  Body:    [JSON schema of response]

Errors:
  [status code] [error code]  -- [when this happens]
  [status code] [error code]  -- [when this happens]
```

**Example:**
```
PATCH /users/:id
Auth:        authenticated (user can only update their own account; admin can update any)
Rate limit:  20 requests per minute per user

Request:
  Params:  id: UUID (required) -- the user's ID
  Query:   none
  Body:    {
    first_name?: string (max 100 chars),
    last_name?:  string (max 100 chars),
    bio?:        string (max 500 chars),
    avatar_url?: string (must be valid URL)
  }
  Note: at least one field must be present

Success:
  Status: 200
  Body:   { data: { id, email, first_name, last_name, bio, avatar_url, updated_at } }

Errors:
  400 VALIDATION_ERROR   -- body fails validation (return details per field)
  401 UNAUTHORIZED       -- not authenticated
  403 FORBIDDEN          -- authenticated but trying to update another user's account
  404 NOT_FOUND          -- user ID does not exist
  422 UNPROCESSABLE      -- avatar_url not reachable or not an image
```

---

### Step 3: API versioning from day one

Version your API in the URL path from the very first endpoint:
```
/api/v1/users
/api/v1/organizations
```

**Rules:**
- Never break a versioned API. Additive changes are OK. Removals and renames require a new version.
- When you release v2, keep v1 running for at least 6 months with a deprecation notice.
- Add `Deprecation` and `Sunset` headers to v1 responses when v2 exists.

**Deprecation header format:**
```
Deprecation: true
Sunset: Wed, 01 Jan 2026 00:00:00 GMT
Link: <https://api.yourapp.com/api/v2/users>; rel="successor-version"
```

---

### Step 4: Consistent error response format

Every error response must have the same shape. Inconsistency forces frontend developers
to write different error-handling logic for every endpoint.

**Standard error shape:**
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed.",
    "details": [
      { "field": "email", "message": "Must be a valid email address" },
      { "field": "password", "message": "Must be at least 8 characters" }
    ],
    "requestId": "req_abc123"
  }
}
```

**Error code conventions:**
- All caps, underscore-separated
- Specific enough to handle programmatically
- Examples: `VALIDATION_ERROR`, `UNAUTHORIZED`, `FORBIDDEN`, `NOT_FOUND`, `CONFLICT`,
  `RATE_LIMIT_EXCEEDED`, `TOKEN_EXPIRED`, `TOKEN_INVALID`, `EMAIL_NOT_VERIFIED`,
  `INSUFFICIENT_SUBSCRIPTION`, `PAYMENT_REQUIRED`, `SERVICE_UNAVAILABLE`

**HTTP status code mapping:**
```
400 Bad Request         -- client sent invalid data (validation failure)
401 Unauthorized        -- not authenticated (no token or invalid token)
403 Forbidden           -- authenticated but lacks permission
404 Not Found           -- resource does not exist
409 Conflict            -- action conflicts with current state (e.g., email already exists)
422 Unprocessable       -- syntactically valid but semantically wrong (e.g., invalid URL)
429 Too Many Requests   -- rate limit exceeded
500 Internal Server Error -- unexpected server error (never leak details)
503 Service Unavailable -- dependency is down
```

---

### Step 5: Pagination design

Never return unbounded lists. All list endpoints MUST be paginated.

**When to use offset pagination:**
- Small to medium datasets (< 1 million rows)
- When users need to jump to a specific page ("Go to page 5")
- When total count is needed for a "X of Y results" display

```
GET /users?page=2&limit=20

Response:
{
  "data": [...],
  "meta": {
    "page": 2,
    "limit": 20,
    "total": 157,
    "totalPages": 8
  }
}
```

**When to use cursor pagination:**
- Large datasets or real-time feeds (rows can be inserted between pages)
- Infinite scroll patterns
- When skipping rows is expensive (very large OFFSET in SQL)

```
GET /events?cursor=eyJpZCI6MTIzfQ&limit=20

Response:
{
  "data": [...],
  "meta": {
    "nextCursor": "eyJpZCI6MTQzfQ",  -- null if last page
    "hasMore": true
  }
}
```

Cursor is a base64-encoded JSON of the last item's sort key. Never expose raw DB IDs directly.

---

### Step 6: Filtering and sorting

Use query parameters consistently:

```
Filtering:
GET /users?status=active&role=admin
GET /users?created_after=2024-01-01&created_before=2024-12-31

Sorting (prefix - for descending):
GET /users?sort=-created_at          -- descending by created_at
GET /users?sort=last_name            -- ascending by last_name
GET /users?sort=last_name,-created_at -- multiple sort keys

Field selection (reduce response size):
GET /users?fields=id,email,first_name
```

Whitelist allowed filter and sort fields server-side. Never pass user-provided column names
directly to SQL (injection risk).

---

### Step 7: API security checklist before writing the first route

Review these before implementing:

| Security concern | What to do |
|---|---|
| Authentication | Every non-public endpoint needs auth middleware |
| Authorization | Auth is not authZ. Verify the user owns or has access to the specific resource |
| Input validation | Validate and sanitize all inputs at the API boundary |
| Rate limiting | Apply to every endpoint. Stricter limits on auth, password reset, signup |
| CORS | Whitelist allowed origins explicitly. Never `*` in production |
| Security headers | CSP, HSTS, X-Frame-Options, X-Content-Type-Options (use helmet) |
| Request size | Limit body size (default Hono/Express limit is 100kb -- reduce for APIs that don't expect large bodies) |
| Error messages | Never return stack traces, DB errors, or file paths to the client |
| SQL injection | Use parameterized queries always. Never concatenate user input into SQL. |
| No sensitive data in URLs | Tokens, passwords, and secrets must go in the request body or headers, never the URL (URL logs) |

---

### Step 8: API documentation format

An API is only as good as its documentation. Every endpoint must be documented so a
frontend developer can build against it without asking a single question.

Include for each endpoint:
1. What it does (one sentence)
2. Auth requirement
3. Full request schema (with examples)
4. Full success response (with example JSON)
5. All error responses (with example JSON)
6. Rate limit information
7. Any side effects (sends email, charges card, etc.)

Use OpenAPI 3.0 annotations in code or write a `docs/api/openapi.yaml`. Both produce
documentation that tools like Swagger UI and Postman can import.

---

## What to avoid

DO NOT name endpoints with verbs: no `/getUser`, `/createOrder`, `/processPayment`.

DO NOT leave the error response format inconsistent across endpoints.

DO NOT return unbounded lists from any endpoint. Paginate everything.

DO NOT pass user-provided field names directly to ORDER BY, WHERE, or SELECT clauses.

DO NOT return different response shapes from the same endpoint based on some condition.

DO NOT nest resources more than 2 levels deep.

DO NOT expose internal IDs or implementation details in the API (DB auto-increment IDs, table names, etc.).

DO NOT skip the security checklist before implementing the first route.

---

## Checklist

Before writing any route handler:

- [ ] All resources are named as nouns (plural)
- [ ] API is versioned at `/api/v1/`
- [ ] Every endpoint has a complete contract (method, path, auth, request schema, response schema, all error codes)
- [ ] Error response format is consistent: always `{ error: { code, message, details, requestId } }`
- [ ] All list endpoints have pagination defined (offset or cursor, documented)
- [ ] Filtering and sorting use query params with server-side whitelisting
- [ ] Auth middleware is planned for every non-public endpoint
- [ ] Authorization logic (ownership checks) is planned, not just authentication
- [ ] Rate limiting is planned for every endpoint with stricter limits for auth routes
- [ ] CORS is configured with explicit allowed origins
- [ ] Security headers middleware is applied
- [ ] Request body size limits are configured
- [ ] No stack traces or internal errors will be returned to the client
- [ ] OpenAPI/documentation annotations are planned alongside implementation
