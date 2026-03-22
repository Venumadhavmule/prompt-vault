# Authentication Skill -- Instruction Set

> Load this skill at the START of any task involving auth: login, registration,
> session management, OAuth, RBAC, JWT, cookies, password handling, or account
> security. Do not proceed without reading all sections.

---

## Prime Directive

Authentication is the highest-risk surface in any application.
Every decision here has a direct security consequence.
You do not improvise. You do not shortcut. You follow proven patterns exactly.
When in doubt, choose the MORE SECURE option, never the more convenient one.

---

## 1. Pre-Work: Identify What Type of Auth Is Needed

Before writing a single line, answer these questions:

| Question | Answer drives |
|---|---|
| Is this a web app, mobile app, or API only? | Cookie strategy (httpOnly for web, bearer for API/mobile) |
| Does it need social login (Google, GitHub)? | OAuth 2.0 / OIDC integration required |
| Does it need multi-tenancy? | Org context must be part of the auth layer |
| Does it need role-based access? | RBAC must be designed at the schema level |
| Is this B2B SaaS? | SSO (SAML/OIDC) may be required |
| Does it need MFA? | TOTP or SMS flow must be planned upfront |

Do not start implementing until all answers are confirmed.
State your assumptions explicitly before proceeding.

---

## 2. Architecture Decisions -- Make These Explicitly

### Token Strategy

DECISION RULE: Access token + Refresh token rotation is the ONLY acceptable pattern for production systems.

```
Access Token:   Short-lived (15 minutes), signed JWT, in-memory on client
Refresh Token:  Long-lived (7-30 days), opaque random string, httpOnly cookie (web) or secure storage (mobile)
                Stored HASHED in DB, one-to-one with a device/session
```

REJECT any pattern that:
- Stores access tokens in localStorage (XSS risk)
- Uses a single long-lived JWT with no refresh
- Stores refresh tokens in plain text in DB

### Session vs JWT

Use SESSIONS when:
- You need instant revocation (compromised accounts, logout from all devices)
- Your app is SSR-only with no mobile API
- You are using Auth.js / NextAuth

Use JWT when:
- You have a separate API server and web client
- You are building a mobile or third-party API
- You need stateless horizontal scaling

NEVER mix both patterns in the same app without a clear boundary.

### Cookie Configuration (Web Only)

Every cookie used for auth MUST have ALL of these flags:
- `httpOnly: true` -- never readable by JavaScript
- `secure: true` -- HTTPS only (enforce in production)
- `sameSite: 'lax'` -- CSRF protection (use 'strict' for maximum protection, 'none' only if cross-origin is required)
- `path: '/'` -- available to all routes
- `maxAge` or `expires` -- never indefinite

---

## 3. Password Security Rules -- Non-Negotiable

These are absolute rules. You do not negotiate or omit any of them.

1. ALWAYS hash with bcrypt, argon2id, or scrypt. Never SHA/MD5/AES for passwords.
2. bcrypt minimum rounds: 12 in production, 10 in test (never lower)
3. ALWAYS validate password strength before hashing: minimum 8 characters, must contain mixed case + number + symbol
4. NEVER log passwords. NEVER send passwords in plain text over any channel.
5. NEVER store password reset tokens in plain text -- store the SHA-256 hash, send the raw token in the email
6. Password reset tokens MUST expire in 1 hour maximum
7. Email verification tokens: store hashed, expire in 24 hours
8. On failed login: return generic message ("Invalid email or password") -- never reveal which field is wrong

---

## 4. Step-by-Step Implementation Order

Follow this order exactly. Do not skip or reorder steps.

### Step 1: Database Schema First

Define these tables BEFORE writing any service code:

```
users
  id (uuid, primary key)
  email (unique, indexed, lowercase)
  password_hash (nullable -- null if OAuth-only account)
  email_verified_at (nullable timestamp)
  role (enum: user, admin, super_admin)
  is_active (boolean, default true)
  created_at, updated_at

refresh_tokens
  id (uuid, primary key)
  user_id (FK -> users.id, indexed)
  token_hash (unique, indexed)
  device_info (text, nullable)
  ip_address (inet)
  expires_at (timestamp, indexed)
  revoked_at (nullable timestamp)
  created_at

audit_logs
  id (uuid, primary key)
  user_id (FK -> users.id, nullable -- null for anonymous events)
  event_type (enum: LOGIN_SUCCESS, LOGIN_FAILURE, LOGOUT, PASSWORD_CHANGE, etc.)
  ip_address (inet)
  user_agent (text)
  metadata (jsonb)
  created_at (indexed)
```

### Step 2: Environment Variables

Ensure these are in .env.example BEFORE writing any code:

```
JWT_ACCESS_SECRET=     # min 32 char random string
JWT_REFRESH_SECRET=    # different from ACCESS, min 32 char
JWT_ACCESS_EXPIRY=15m
JWT_REFRESH_EXPIRY=7d
BCRYPT_ROUNDS=12
```

### Step 3: Implement Utility Functions First

Order: hash password -> verify password -> generate access token -> verify access token -> generate refresh token -> validate refresh token

Each utility function MUST:
- Be a pure function with no DB calls
- Have its own unit test
- Never throw unhandled exceptions -- wrap errors and rethrow typed errors

### Step 4: Implement AuthService

Methods in this order:
1. `register(email, password)` -- create user, send verification email, return tokens
2. `login(email, password)` -- verify, check active status, rotate refresh, return tokens
3. `refresh(refreshToken)` -- detect reuse, rotate token, return new access token
4. `logout(refreshToken)` -- revoke specific token
5. `logoutAll(userId)` -- revoke all tokens for user
6. `verifyEmail(token)` -- mark email verified
7. `forgotPassword(email)` -- generate reset token, send email
8. `resetPassword(token, newPassword)` -- validate token, update password, revoke all tokens
9. `changePassword(userId, currentPassword, newPassword)` -- verify current, update

### Step 5: Implement Middleware

Create these middleware in order:
1. `requireAuth` -- verify JWT, attach user to request context, throw 401 on failure
2. `optionalAuth` -- attempt JWT verification, attach user if found, never throw
3. `requireRole(roles[])` -- check user has required role, throw 403 on failure
4. `requirePermission(permission)` -- check specific permission, throw 403 on failure

### Step 6: Apply Rate Limiting to Auth Routes

Auth routes MUST have stricter rate limits than all other routes:
- Login: 5 attempts per 15 minutes per IP
- Register: 3 attempts per hour per IP
- Password reset request: 3 attempts per hour per email
- Email verification resend: 3 per hour per user

Use Redis (leaky bucket or sliding window) if available.
Fall back to in-memory rate limiting only in dev/test.

### Step 7: Apply Routes

Route pattern for auth:
```
POST /auth/register
POST /auth/login
POST /auth/logout
POST /auth/refresh
POST /auth/forgot-password
POST /auth/reset-password
POST /auth/verify-email
POST /auth/resend-verification
GET  /auth/me  (requireAuth)
```

### Step 8: Write Tests

REQUIRED test cases -- do not ship without all of these:

Registration:
- Happy path: valid email + password -> user created, tokens returned, verification email queued
- Duplicate email -> 409 Conflict
- Invalid email format -> 400 Validation Error
- Weak password -> 400 Validation Error with specific reason

Login:
- Happy path -> tokens returned, audit log written
- Wrong password -> 401 (generic message)
- Account not found -> 401 (same generic message -- no information leakage)
- Unverified email -> 403 with specific error code
- Inactive account -> 403 with specific error code
- Rate limited after 5 failures -> 429

Token Refresh:
- Valid refresh token -> new access token returned, refresh token rotated
- Expired refresh token -> 401
- Revoked refresh token -> 401 + LOG reuse detection + alert
- Refresh token from different user -> 401

Password Reset:
- Valid reset flow (request + reset) -> password updated, all sessions revoked
- Expired token -> 400
- Already-used token -> 400
- Reset to same password -> 400 with specific message

---

## 5. RBAC Implementation Rules

### Define Roles and Permissions Explicitly

NEVER check roles with raw string comparison in business logic.
Define an enum or const object for all roles AND a permission matrix.

Structure:
```
Roles:       SUPER_ADMIN, ADMIN, USER, VIEWER, GUEST
Permissions: READ_OWN_DATA, WRITE_OWN_DATA, READ_ALL_DATA, MANAGE_USERS, etc.

Role -> Permission mapping (static, code-defined, not DB-driven unless you need dynamic RBAC)
```

### Permission Check Rules

Always check PERMISSION, not ROLE in business logic:
- CORRECT: `if (!hasPermission(user, 'DELETE_MEMBER'))` 
- WRONG: `if (user.role !== 'ADMIN')`

This allows role definitions to change without touching business logic.

Resource ownership check: ALWAYS verify the resource belongs to the requesting user unless the user has elevated permissions.

---

## 6. OAuth / Social Login Rules

When implementing OAuth (Google, GitHub, etc.):

1. NEVER store OAuth access tokens long-term. Use them once to get the user profile, then discard.
2. Link by email with explicit user consent if an account already exists with that email.
3. Always verify the OAuth provider's ID token (use the provider's public key / SDK, never parse raw).
4. Store `provider` and `providerId` on the user or a separate `oauth_accounts` table.
5. Handle the case where a user revokes OAuth access -- degrade gracefully.

---

## 7. Security Checklist -- Run Before Marking Any Auth Task Complete

- [ ] Passwords hashed with bcrypt/argon2 (rounds >= 12)
- [ ] JWT secrets loaded from env, not hardcoded
- [ ] Access token expiry <= 15 minutes
- [ ] Refresh tokens stored HASHED in DB
- [ ] httpOnly cookies used for refresh tokens in web apps
- [ ] Rate limiting applied to all auth endpoints
- [ ] Generic error messages (no information leakage)
- [ ] CSRF protection in place (SameSite cookie or CSRF token)
- [ ] Email verification enforced before sensitive actions
- [ ] Audit log written for every auth event
- [ ] All refresh tokens revoked on password change
- [ ] Refresh token reuse detection implemented and logged
- [ ] SQL injection not possible (parameterized queries only)
- [ ] No secrets or tokens in server logs

---

## 8. Common Mistakes -- Catch and Fix Immediately

| Mistake | Why It's Wrong | Correct Pattern |
|---|---|---|
| `jwt.verify()` without catching errors | Throws on invalid token, crashes request | Always wrap in try/catch, return 401 |
| Comparing tokens directly from DB | Timing attack vulnerability | Use constant-time comparison (`crypto.timingSafeEqual`) |
| Sending userId in refresh token body | Tokens should be opaque; look up userId from DB | Store token hash in DB, look up on use |
| `setTimeout` for token cleanup | Unreliable, memory leak risk | Use DB expiry + cron cleanup job |
| Returning `200 OK` on failed login | Client-side confusion | Always `401 Unauthorized` |
| Not rate-limiting per IP AND per account | Both vectors need protection | Implement both dimensions |
| Storing JWTs in localStorage | XSS can steal them | Use httpOnly cookies or in-memory only |

---

## 9. Cross-References

- Error types used: `UnauthorizedError`, `ForbiddenError`, `ValidationError`, `RateLimitError` -> see `00_error_system`
- Environment variables -> see `00_environment_config`
- Audit log schema -> see `02_database_skill`
- Rate limiting middleware -> see `03_api_design_skill`
- Testing patterns -> see `05_testing_skill`
