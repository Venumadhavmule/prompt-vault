# WF03 -- Auth Implementation Workflow

## Purpose

Step-by-step runbook for implementing JWT-based authentication with access/refresh
tokens from scratch. Follow in order. Do not skip the schema step to go straight
to routes.

---

## Pre-conditions

- Database is connected and Prisma is set up
- Environment variables file exists (`.env`)
- Decision made on auth method: this workflow covers JWT with httpOnly refresh token cookie

---

## Step 1: Define the schema

Add these tables to `prisma/schema.prisma`:

```prisma
model User {
  id              String    @id @default(cuid())
  email           String    @unique
  password_hash   String
  first_name      String
  last_name       String
  email_verified  Boolean   @default(false)
  created_at      DateTime  @default(now())
  updated_at      DateTime  @updatedAt

  refresh_tokens  RefreshToken[]
}

model RefreshToken {
  id          String    @id @default(cuid())
  token       String    @unique
  user_id     String
  expires_at  DateTime
  revoked     Boolean   @default(false)
  created_at  DateTime  @default(now())

  user  User  @relation(fields: [user_id], references: [id], onDelete: Cascade)

  @@index([user_id])
  @@index([token])
}
```

---

## Step 2: Create the migration

```bash
npx prisma migrate dev --name add_auth_tables
```

Verify the generated SQL:
- `user` table created with all columns
- `refresh_token` table created with FK constraint and ON DELETE CASCADE
- Indexes on `refresh_token.token` and `refresh_token.user_id`

---

## Step 3: Install dependencies

```bash
pnpm add bcryptjs jsonwebtoken
pnpm add -D @types/bcryptjs @types/jsonwebtoken
```

---

## Step 4: Add environment variables

In `.env`:
```
JWT_ACCESS_SECRET=<generate with: openssl rand -hex 32>
JWT_REFRESH_SECRET=<generate with: openssl rand -hex 32>
ACCESS_TOKEN_EXPIRY=900        # 15 minutes in seconds
REFRESH_TOKEN_EXPIRY=604800    # 7 days in seconds
```

In `.env.example`:
```
JWT_ACCESS_SECRET=your_access_secret_here
JWT_REFRESH_SECRET=your_refresh_secret_here
ACCESS_TOKEN_EXPIRY=900
REFRESH_TOKEN_EXPIRY=604800
```

---

## Step 5: Implement the auth service

Create `src/services/authService.ts`:

Functions to implement:
1. `hashPassword(password: string): Promise<string>` -- bcryptjs, cost factor 12
2. `verifyPassword(password: string, hash: string): Promise<boolean>`
3. `generateAccessToken(userId: string): string` -- signed with JWT_ACCESS_SECRET, expiry from env
4. `generateRefreshToken(): string` -- crypto.randomUUID() or randomBytes
5. `register(email, password, firstName, lastName): Promise<User>` -- hash password, create user
6. `login(email, password): Promise<{accessToken, refreshToken, user}>` -- verify credentials, create refresh token record, return tokens
7. `refresh(refreshToken): Promise<{accessToken, refreshToken}>` -- verify token validity, revoke old, issue new (rotation)
8. `logout(refreshToken): Promise<void>` -- revoke the token in DB

Reference: `skills/01_authentication_skill.md` for full implementation details.

---

## Step 6: Implement the auth routes

Create `src/routes/auth.ts` (or `src/app/api/auth/[...]/route.ts` for Next.js):

Required endpoints:

| Method | Path | Purpose |
|--------|------|---------|
| POST | /api/auth/register | Create account |
| POST | /api/auth/login | Exchange credentials for tokens |
| POST | /api/auth/refresh | Exchange refresh token for new access token |
| POST | /api/auth/logout | Revoke refresh token |

For each endpoint:
- Validate input with Zod (or your validation library)
- Call the auth service
- Set refresh token as httpOnly cookie
- Return access token in response body (not in cookie)

---

## Step 7: Implement auth middleware

Create `src/middleware/auth.ts`:

```
Function: authenticateRequest(req) => { userId: string }
- Extract Bearer token from Authorization header
- Verify with jwt.verify() using JWT_ACCESS_SECRET
- Return the decoded userId
- Throw 401 if missing, expired, or invalid
```

Apply this middleware to all routes that require authentication.

---

## Step 8: Implement the auth repository

Create `src/repositories/authRepository.ts`:

Functions:
1. `findUserByEmail(email: string): Promise<User | null>`
2. `createUser(data): Promise<User>`
3. `saveRefreshToken(userId, token, expiresAt): Promise<void>`
4. `findRefreshToken(token): Promise<RefreshToken | null>`
5. `revokeRefreshToken(token): Promise<void>`
6. `revokeAllUserTokens(userId): Promise<void>`

---

## Step 9: Write tests

Tests to write (in order):

1. `authService.test.ts` -- unit tests with mocked repository
   - [ ] hashPassword produces a valid bcrypt hash
   - [ ] verifyPassword returns true for correct password
   - [ ] verifyPassword returns false for wrong password
   - [ ] login throws INVALID_CREDENTIALS for wrong password
   - [ ] login throws INVALID_CREDENTIALS for unknown email
   - [ ] login returns tokens for valid credentials
   - [ ] refresh returns new tokens
   - [ ] refresh throws if token is revoked
   - [ ] refresh throws if token is expired

2. `auth.routes.test.ts` -- integration tests (real HTTP, test database)
   - [ ] POST /register creates user and returns 201
   - [ ] POST /register returns 409 for duplicate email
   - [ ] POST /login returns 200 with access_token and sets cookie
   - [ ] POST /login returns 401 for wrong password
   - [ ] POST /refresh returns 200 with new access_token
   - [ ] POST /refresh returns 401 for missing or invalid cookie
   - [ ] POST /logout returns 200 and invalidates the token
   - [ ] Protected route returns 401 without token
   - [ ] Protected route returns 200 with valid token

---

## Step 10: Security verification

Before shipping, verify every item:

- [ ] Passwords hashed with bcrypt cost factor 12+
- [ ] Access token expiry is 15 minutes or less
- [ ] Refresh tokens are stored hashed or as opaque values -- not the raw JWT
- [ ] Refresh token rotation is implemented (old token revoked when new one issued)
- [ ] httpOnly + SameSite=Strict (or Lax) on refresh token cookie
- [ ] HTTPS only for cookie (`Secure` attribute in production)
- [ ] No user enumeration: both "email not found" and "wrong password" return the same error message
- [ ] Rate limiting on /login endpoint
- [ ] All auth tokens are purged on logout (not just removed from client)

Reference: `skills/40_quality_security_review.md`

---

## Done definition

- All tests from Step 9 pass
- Security checklist from Step 10 is complete
- Manual test: register a user, log in, call a protected route, refresh token, log out, confirm refreshed token is invalid
- No secrets committed to version control
