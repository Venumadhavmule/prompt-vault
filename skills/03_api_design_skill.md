# API Design Skill -- Instruction Set

> Load this skill at the START of any task involving REST API design,
> route handlers, request validation, response formatting, middleware,
> versioning, security headers, rate limiting, or API documentation.

---

## Prime Directive

An API is a contract. Once clients depend on it, breaking changes cause real damage.
Design intentionally. Validate everything at the boundary. Format consistently.
Security and correctness come before convenience.

---

## 1. Pre-Work: API Design Before Code

Before writing any route handler, define:

1. The resource being operated on (users, orders, subscriptions)
2. The HTTP method and what it semantically means for this resource
3. The request schema: what body/query/params are accepted
4. The success response shape: what is returned and with what status code
5. The error responses: what can go wrong and what status code each returns
6. Authentication requirement: public, authenticated user, or specific role
7. Rate limiting requirements: is this a sensitive or high-cost operation?

Do NOT write code until all 7 are answered.

---

## 2. HTTP Method Semantics -- Follow These Exactly

| Operation | Method | Idempotent | Body | Status on Success |
|---|---|---|---|---|
| Fetch resource | GET | Yes | No | 200 |
| Fetch list | GET | Yes | No | 200 |
| Create resource | POST | No | Yes | 201 |
| Full replace | PUT | Yes | Yes | 200 |
| Partial update | PATCH | No | Yes | 200 |
| Delete resource | DELETE | Yes | No | 204 |
| Trigger action | POST | No | Yes | 200 or 202 |

Rules:
- GET requests MUST NEVER cause side effects. Do not use GET for anything that modifies data.
- PUT is a full replace -- the body contains the complete new state of the resource
- PATCH is a partial update -- only the provided fields are changed
- DELETE returns 204 No Content on success (no body)
- 202 Accepted: use when the action is queued for async processing

---

## 3. URL Design Rules

### Resource Naming

- Use NOUNS for resources, never verbs
- Plural nouns for collections
- Lowercase, kebab-case only, no underscores

```
CORRECT: /users, /subscription-plans, /payment-methods
WRONG:   /getUsers, /subscriptionPlans, /payment_methods
```

### URL Structure

```
Collection:         /api/v1/users
Single resource:    /api/v1/users/:userId
Nested resource:    /api/v1/users/:userId/orders
Action on resource: /api/v1/users/:userId/verify-email  (POST)
```

### Nesting Limit

Never nest more than 2 levels deep. If you need deeper nesting, restructure the resource:
```
WRONG:  /organizations/:orgId/teams/:teamId/members/:memberId/permissions
CORRECT: /team-members/:memberId/permissions (memberId globally unique)
```

---

## 4. Request Validation Rules

EVERY route that accepts a body, query string, or path parameter MUST validate the input.

Validation MUST happen before any business logic or database calls.

### Validation Layer Pattern

1. Define a Zod schema for the request (body, query, params separately)
2. Apply a validation middleware that parses and validates the request
3. On validation failure: return 400 with a structured error showing which fields failed and why
4. After validation: the handler receives a typed, validated request object

### Validation Error Response Shape

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      { "field": "email", "message": "Invalid email format" },
      { "field": "password", "message": "Must be at least 8 characters" }
    ],
    "requestId": "abc123"
  }
}
```

### What to Validate

| Input | Rules |
|---|---|
| email | Format validation + lowercase transform |
| password | Length + complexity (strength check in auth context) |
| IDs (path params) | UUID format validation |
| Pagination (limit) | Integer, min 1, max 100, default 20 |
| Pagination (page/cursor) | Integer or string, no negative values |
| Dates | ISO 8601 format, coerce to Date object |
| Enums | Strict enum validation (reject unknown values) |
| Strings | `.trim()` applied, max length enforced |
| File names / paths | Sanitize before use, reject path traversal patterns |

---

## 5. Response Formatting Rules

Every API response MUST follow a consistent shape. Never return ad-hoc shapes.

### Success Response (Single Resource)

```json
{
  "data": { ...resource fields... },
  "meta": {
    "requestId": "uuid"
  }
}
```

### Success Response (List)

```json
{
  "data": [ ...items... ],
  "meta": {
    "total": 143,
    "page": 1,
    "limit": 20,
    "hasMore": true,
    "requestId": "uuid"
  }
}
```

Or for cursor pagination:
```json
{
  "data": [ ...items... ],
  "meta": {
    "nextCursor": "encoded_cursor_value",
    "hasMore": true,
    "requestId": "uuid"
  }
}
```

### Error Response

```json
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "User not found",
    "requestId": "uuid"
  }
}
```

Rules:
- NEVER return just a raw string or number as a response body
- NEVER include stack traces in error responses
- ALWAYS include `requestId` in every response
- NEVER change the top-level shape between success and error (always `data` or `error`)

---

## 6. HTTP Status Code Reference

Use these exact status codes. Do not invent new ones or repurpose existing ones.

| Status | When to Use |
|---|---|
| 200 OK | Successful GET, PATCH, PUT, or action |
| 201 Created | Successful POST that created a new resource |
| 202 Accepted | Request accepted, processing is async |
| 204 No Content | Successful DELETE |
| 400 Bad Request | Validation failure, malformed request |
| 401 Unauthorized | Missing or invalid authentication |
| 403 Forbidden | Authenticated but not authorized for this resource |
| 404 Not Found | Resource does not exist |
| 409 Conflict | Duplicate resource, state conflict |
| 422 Unprocessable | Semantically invalid (valid format, invalid business rule) |
| 429 Too Many Requests | Rate limit exceeded |
| 500 Internal Server Error | Unhandled server error |
| 503 Service Unavailable | DB or external service is down |

Rule: NEVER return 200 for an error condition. The HTTP status code conveys the outcome.

---

## 7. Middleware Stack -- Apply in This Order

Every route MUST have middleware applied in this exact order:

```
1. Request ID        -- attach UUID to request before anything else
2. Request Logger    -- log method, path, start time
3. Security Headers  -- CORS, CSP, HSTS, X-Frame-Options, etc.
4. Body Parser       -- parse JSON body (with size limit)
5. Rate Limiter      -- global per-IP rate limiting
6. Auth Middleware   -- verify JWT if route requires auth
7. Route-specific    -- auth guards, resource-level rate limits
8. Validation        -- validate request body/query/params
9. Route Handler     -- business logic
10. Error Handler    -- catch all errors, format and respond
11. Response Logger  -- log status code and duration
```

Missing any of these for a production route is a bug.

---

## 8. Security Headers -- Always Applied

Apply all of these to every HTTP response:

```
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 0  (deprecated, but set to 0 to disable broken browser behavior)
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
Content-Security-Policy: default-src 'self'; ...
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

Use `helmet` (Express), `@hono/secure-headers` (Hono), or equivalent.
Never disable security headers in production.

---

## 9. CORS Configuration Rules

NEVER use `origin: '*'` (wildcard) in production.
NEVER use `credentials: true` with a wildcard origin (this combination is rejected by browsers AND is a security vulnerability).

```
Allowed origins: explicit list from environment variable
Credentials:     true only if your app uses cross-origin cookies
Methods:         GET, POST, PUT, PATCH, DELETE, OPTIONS
Headers:         Content-Type, Authorization, X-Request-ID
Max age:         86400 (1 day for preflight cache)
```

---

## 10. Rate Limiting Strategy

Layer your rate limiting:

| Layer | Target | Limit |
|---|---|---|
| Global | Per IP | 100 req / 15 min |
| API-wide | Per authenticated user | 1000 req / hour |
| Write operations | Per user | 60 req / min |
| Auth routes | Per IP | 5 req / 15 min |
| Password reset | Per email | 3 req / hour |
| File upload | Per user | 10 req / hour |

Rate limit response MUST include:
- `Retry-After` header (seconds until limit resets)
- `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` headers
- 429 status code

---

## 11. API Versioning Strategy

Use URL versioning for REST APIs:
```
/api/v1/users
/api/v2/users  (when breaking changes required)
```

Rules:
- Never change a v1 endpoint in a breaking way -- add v2
- Keep v1 alive for at minimum 6 months after v2 launches
- Add `Deprecation` and `Sunset` headers to deprecated endpoints
- Document the migration path in the changelog

Breaking change definition: any change that requires clients to update their code:
- Removing a field from response
- Renaming a field
- Changing a field type
- Changing required validation rules
- Changing error codes / status codes

---

## 12. Request/Response Logging Rules

Log this for EVERY request:
```
timestamp, requestId, method, path, statusCode, durationMs, userId (if authed), ip
```

NEVER log:
- Passwords (in any field, including logs of the request body)
- JWT tokens or cookies
- API keys or secrets
- Credit card data or PAN
- Full Authorization headers

Implement automatic redaction of these fields in your logger configuration.
Review log output in dev mode to verify redaction before deploying.

---

## 13. Health Check and Readiness Routes

Every API service MUST expose:

```
GET /health
-- Returns 200 if service is running at all (even if DB is down)
-- Used by load balancer to decide whether to route traffic

GET /ready
-- Returns 200 only if ALL dependencies (DB, Redis, etc.) are healthy
-- Returns 503 if any dependency is down
-- Used by Kubernetes readiness probe

Response: {
  "status": "ok" | "degraded" | "down",
  "version": "1.2.3",
  "uptime": 3600,
  "dependencies": {
    "database": { "status": "ok", "latency_ms": 2 },
    "redis": { "status": "ok", "latency_ms": 1 }
  }
}
```

---

## 14. What to NEVER Do

- Never trust client-provided IDs without verifying ownership (e.g., `?userId=` in query)
- Never return unbounded lists -- always paginate
- Never expose DB IDs in sequential integer form (use UUID or CUID2)
- Never log full request bodies in production (PII risk)
- Never allow CORS from all origins in production
- Never return 200 for an error
- Never skip validation on any input, even "internal" endpoints
- Never forward raw error messages from the DB or external services to the client

---

## 15. Cross-References

- Error types and global handler -> see `00_error_system`
- Auth middleware -> see `01_authentication_skill`
- DB repository pattern -> see `02_database_skill`
- Environment variables -> see `00_environment_config`
- Testing API routes -> see `05_testing_skill`
