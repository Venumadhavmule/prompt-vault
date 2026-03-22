# SKILL 2.4 -- How to Plan Authentication for Any App

## What this skill is

This skill covers the up-front design decisions for authentication in any application.
Auth is the most replayed mistake in software engineering -- developers jump straight
to implementation without a design, then discover mid-project that their token strategy
is wrong, their permission model is too simple, or their session invalidation is
broken. This skill ensures you answer the hard questions before writing a line of code.

This is a planning skill. For implementation, use `31_execution_auth.md`.

---

## When to use this skill

- At the start of any project that has users
- When adding auth to an existing app
- When redesigning auth due to security requirements
- Before choosing an auth library or service
- When planning for compliance (GDPR, SOC2, HIPAA)

---

## Full Guide

### Step 1: Identify the auth type your app requires

Answer these questions before picking any implementation approach:

| Question | Why it matters |
|---|---|
| Is this a web app, mobile app, or API only? | Determines cookie vs bearer token strategy |
| Do users sign up with email/password, social login, or both? | Determines OAuth integration scope |
| Are there corporate/enterprise buyers who need SSO? | SAML/OIDC required for enterprise |
| Does the app have multiple user types with different permissions? | RBAC must be designed at schema level |
| Is MFA required? | TOTP or SMS flow must be planned upfront |
| Is this multi-tenant (multiple organizations)? | Org context must be part of the auth layer |
| What compliance standards apply? (HIPAA, SOC2, PCI) | Determines audit log requirements, session policy, encryption |

---

### Step 2: Choose the auth method

**Decision tree:**

```
Is this an API consumed by machines (no human login)?
  YES → Use API keys (long-lived, scoped, rotatable)
  NO  ↓

Is this a web app only (no mobile, no external API clients)?
  YES → JWT access token (memory) + refresh token (httpOnly cookie)
  NO  ↓

Is this a web + mobile + API (multiple client types)?
  YES → JWT bearer token flow with refresh. Mobile: secure storage for refresh token.
  NO  ↓

Is this a B2B app where enterprise companies need to connect their SSO?
  YES → OAuth 2.0 / OIDC as the primary flow. Support SAML for legacy enterprise IdPs.
  NO  ↓

Default: JWT access token + refresh token rotation
```

**Auth method comparison:**

| Method | Best for | Limitations |
|---|---|---|
| API key | Machine-to-machine, developer APIs | No built-in expiry, must be stored securely |
| Session + cookie | Simple web apps, no mobile | Hard to scale horizontally without shared session store |
| JWT (stateless) | APIs, mobile, SPAs | Cannot invalidate before expiry without a blocklist |
| JWT + refresh rotation | Most SaaS apps | Requires refresh token table in DB |
| OAuth 2.0 / OIDC | Social login, federated identity | Complexity overhead |
| SAML | Enterprise SSO (legacy IdPs like Okta, ADFS) | XML-heavy, complex to implement |

**Default recommendation for SaaS:**
Access token (JWT, 15 min) + Refresh token (opaque UUID, 30 days, httpOnly cookie for web).

---

### Step 3: Plan the token lifecycle

For each token type, define all of the following:

**Access Token:**
```
Type:         Signed JWT
Algorithm:    HS256 (single service) or RS256 (multiple services that verify)
Expiry:       15 minutes
Contents:     { sub: userId, email, roles, orgId (if multi-tenant), iat, exp }
DO NOT include: password, raw permissions, credit card info, PII beyond what's needed
Storage:      In-memory only on the client (never localStorage, never cookie)
```

**Refresh Token:**
```
Type:         Opaque random string (UUID v4 or crypto.randomBytes(32).toString('hex'))
Expiry:       30 days (sliding if user is active, absolute otherwise)
Storage:      httpOnly, Secure, SameSite=Strict cookie (web)
              Secure device storage (iOS Keychain / Android Keystore) for mobile
DB storage:   Store the HASH (bcrypt or SHA-256) not the raw token
One-per-session: Each active session has its own refresh token row
```

**Refresh token rotation:**
Every time the refresh token is used, issue a new one and invalidate the old one.
If a refresh token is used AFTER it was already rotated (reuse detection), invalidate
ALL tokens in that session family. This detects token theft.

---

### Step 4: Plan the permission model

Choose the right permission model for your use case:

| Model | What it is | When to use |
|---|---|---|
| Simple boolean `is_admin` | Admin vs regular user | Only 2 permission levels exist forever |
| Role-based (RBAC) | Roles assigned to users (admin, member, viewer). Roles map to permissions. | Most SaaS apps with multiple permission levels |
| Attribute-based (ABAC) | Permissions based on attributes of the user, resource, and environment | Complex enterprise systems, fine-grained data access |
| Organization RBAC | RBAC where roles are scoped to an organization | Multi-tenant apps |

**Default recommendation for SaaS:** Organization-scoped RBAC.

**Schema for RBAC:**
```sql
-- Roles: admin, member, viewer (or custom per product)
-- Store role on the membership table, not on the user
organization_memberships (
  user_id    UUID REFERENCES users(id),
  org_id     UUID REFERENCES organizations(id),
  role       TEXT CHECK (role IN ('owner', 'admin', 'member', 'viewer')),
  PRIMARY KEY (user_id, org_id)
)
```

**Permission checks:** Define permissions as code constants, not strings scattered through the codebase:
```
PERMISSIONS = {
  'members:read':  ['owner', 'admin', 'member', 'viewer'],
  'members:write': ['owner', 'admin'],
  'billing:read':  ['owner', 'admin'],
  'billing:write': ['owner'],
  'org:delete':    ['owner']
}
```

---

### Step 5: Plan session invalidation strategy

You must be able to invalidate sessions. Without this, users who are compromised,
deleted, or locked out cannot be forced offline.

**Scenarios to plan for:**

| Scenario | Required capability |
|---|---|
| User clicks logout | Invalidate current session's refresh token only |
| User changes password | Invalidate ALL sessions (force re-login everywhere) |
| User deletes account | Invalidate ALL tokens, mark user as deleted |
| Admin locks an account | Invalidate ALL tokens, block future logins |
| Suspicious activity detected | Invalidate ALL tokens, send security alert email |
| Force logout all devices | Invalidate ALL refresh tokens for the user |

**Implementation:** Maintain a `refresh_tokens` table. On invalidation, set `revoked_at`.
The auth middleware checks if the token is revoked BEFORE accepting it.

For JWT access tokens (stateless), invalidation requires either:
1. Accept that tokens are valid until expiry (15 min window) -- usually fine
2. Maintain a JWT blocklist in Redis (token jti → expiry) -- needed for immediate invalidation

---

### Step 6: Plan the password policy

```
Minimum length:    12 characters (8 is outdated guidance)
Complexity:        Encouraged but not required (long passphrases are stronger than complex short passwords)
Breach detection:  Check against HaveIBeenPwned on registration and password change
bcrypt rounds:     Minimum 12. Test on your hardware: aim for 100-300ms hash time.
Storage:           bcrypt hash only. NEVER: plaintext, MD5, SHA1, SHA256 without salt
Reset token:       Cryptographically random, hashed before DB storage, expires in 60 min, single use
Rate limiting:     Max 5 login attempts per 15 min per IP and per account
Account lockout:   Optional (discuss with product -- lockout can be used for DoS against users)
```

---

### Step 7: Plan for compliance requirements

| Standard | Auth implications |
|---|---|
| GDPR | User must be able to delete account + all tokens. Export of session data may be required. |
| SOC2 | Audit logs of all auth events. MFA enforcement for staff accounts. Access review processes. |
| HIPAA | Automatic session timeout after inactivity (typically 15-30 min). Audit logs mandatory. Strong MFA. |
| PCI DSS | No storing of card data (handled by Stripe). Strong MFA for admin access. |

**Audit log must capture:**
- Login success (userId, IP, userAgent, timestamp)
- Login failure (email attempted, IP, timestamp -- do NOT log the password)
- Logout
- Password change
- Password reset request
- Refresh token reuse detected (security event)
- Account locked
- Role change (who changed whose role to what)

---

### Step 8: Common auth planning mistakes

| Mistake | Consequence |
|---|---|
| Storing access tokens in localStorage | XSS attack steals all user sessions |
| Long-lived JWTs with no refresh | Cannot invalidate compromised tokens |
| Storing refresh tokens in localStorage | Same XSS risk as access tokens |
| Returning same error for wrong email vs wrong password | Prevents email enumeration (good) but MUST be intentional |
| Not hashing the refresh token in DB | DB breach exposes all active sessions |
| Implementing RBAC with hardcoded role checks in business logic | Role model cannot evolve without code changes everywhere |
| No audit log for auth events | Cannot investigate security incidents |
| Not planning for forced logout | Compromised accounts cannot be safely closed |
| Skipping MFA for admin accounts | Admin compromise leads to full system compromise |
| Using the same JWT secret across all environments | Dev/staging token is valid in production |

---

### Auth Design Document Template

```markdown
# Auth Design: [App Name]

## Auth Method
[JWT + refresh rotation | OAuth | API keys | etc.]
Reason: [why this method for this app]

## Token Lifecycle
### Access Token
- Type: JWT | Expiry: 15min
- Signing: HS256 with JWT_SECRET env var
- Claims: [list claims]
- Client storage: in-memory only

### Refresh Token
- Type: opaque UUID | Expiry: 30 days
- Client storage: httpOnly cookie
- DB storage: sha256 hash of token

## Permission Model
- Type: [RBAC | ABAC | simple boolean]
- Roles defined: [list roles and their capabilities]
- Permission definitions: [link to permissions file]

## Session Invalidation
- Logout: revoke current session token
- Password change: revoke all tokens
- Account lock: revoke all tokens, block login

## Password Policy
- Min length: 12
- bcrypt rounds: 12
- Breach detection: Yes (HaveIBeenPwned)
- Reset flow: token expires 60min, single use

## Audit Logging
- Events logged: [list]
- PII masking: [what is masked in logs]

## Compliance
- [Standards being met and specific controls]
```

---

## Checklist

Before implementing any auth system:

- [ ] Auth method chosen and documented (JWT/session/API key/OAuth)
- [ ] Access token: expiry, algorithm, claims, storage location defined
- [ ] Refresh token: expiry, storage (httpOnly cookie), DB storage (hashed) defined
- [ ] Refresh token rotation plan documented
- [ ] Reuse detection plan documented
- [ ] Permission model chosen (RBAC/ABAC) and schema designed
- [ ] Role definitions written as code constants
- [ ] Session invalidation scenarios covered (logout, password change, account lock, all devices)
- [ ] Password policy defined (min length, bcrypt rounds, breach detection)
- [ ] Audit log events listed (no passwords or tokens in logs)
- [ ] Rate limiting plan for auth routes (login, register, forgot-password)
- [ ] Email verification flow designed
- [ ] Password reset flow designed (token expiry, single use, hashed storage)
- [ ] Compliance requirements identified and controls mapped
- [ ] .env variables for auth system listed (JWT_SECRET, REFRESH_TOKEN_EXPIRY, etc.)
- [ ] No access tokens in localStorage -- ever
- [ ] No plaintext tokens stored in DB -- always hash
