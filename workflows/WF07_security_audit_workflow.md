# WF07 -- Security Audit Workflow

## Purpose

Step-by-step runbook for conducting a systematic security review of a codebase.
Use this before every major release, after any auth or billing changes, and
quarterly on production systems. Security issues found before shipping cost
near zero. Security issues found after a breach cost orders of magnitude more.

---

## Pre-conditions

- Access to the full codebase
- Access to dependency manifests (package.json, lockfile)
- Access to environment variable definitions (.env.example)
- Ideally: access to the last 30 days of application logs

---

## Audit sequence: work through each category in order

---

## Category 1: Input Validation and Injection

### SQL Injection
- [ ] All database queries use the ORM (Prisma) or parameterized queries
- [ ] No string concatenation used to build SQL queries
- [ ] Search for: `db.execute(` or `queryRaw` -- review every instance manually

### XSS (Cross-Site Scripting)
- [ ] No raw HTML rendering of user-provided content (`dangerouslySetInnerHTML` in React)
- [ ] All user input in templates is escaped by default
- [ ] Rich text editors use a sanitization library (DOMPurify) before saving and rendering

### Command Injection
- [ ] No `exec()`, `spawn()`, or `child_process` calls with user-controlled input
- [ ] No file path construction using user-supplied strings without sanitization

### Mass Assignment
- [ ] API handlers do not spread the request body directly onto DB update calls
- [ ] Only explicitly allowed fields are passed to create/update operations

---

## Category 2: Authentication and Authorization

- [ ] All protected routes require authentication
- [ ] Authentication middleware is applied to route groups, not individually (no accidental gaps)
- [ ] JWT tokens are verified with the correct algorithm (RS256 or HS256 -- no `alg: none`)
- [ ] Token expiry is enforced (access token ≤ 15 minutes)
- [ ] Refresh token rotation is implemented (old token revoked when new one issued)
- [ ] Password hashing uses bcrypt/argon2 with cost factor ≥ 12
- [ ] Login endpoint has rate limiting
- [ ] Failed login does not reveal whether the email exists

### Authorization (IDOR checks)
This is the most commonly missed vulnerability. For every data-access operation:
- [ ] Does the user own or have permission to access the resource they are requesting?
- [ ] Is the ownership check done in server-side code (not based on a client-provided flag)?
- [ ] For every `findUnique` and `findFirst` call: does the query include `userId` or a team/org filter?

Typical IDOR pattern to catch:
```typescript
// VULNERABLE: any authenticated user can access any user's data
const invoice = await prisma.invoice.findUnique({ where: { id: invoiceId } });

// SECURE: constrains to the authenticated user's invoices only
const invoice = await prisma.invoice.findUnique({
  where: { id: invoiceId, userId: currentUser.id },
});
```

---

## Category 3: Secrets and Sensitive Data Exposure

- [ ] No secrets in source code (API keys, JWT secrets, database passwords)
- [ ] No secrets in git history (check with: `git log --all -S "sk_live_" --oneline`)
- [ ] All secrets are in environment variables
- [ ] `.env` is in `.gitignore`
- [ ] `.env.example` contains only placeholder values, never real keys
- [ ] Passwords are never logged
- [ ] Tokens are never logged
- [ ] PII (email, name, address) is not logged at DEBUG level or below
- [ ] Search for: `console.log(password` `console.log(token` `console.log(user`

---

## Category 4: Dependency Vulnerabilities

```bash
# Run dependency audit
pnpm audit

# For high/critical vulnerabilities only
pnpm audit --severity high

# Check for outdated packages
pnpm outdated
```

- [ ] No high or critical severity vulnerabilities in dependencies
- [ ] Any known-vulnerable package has either been updated or has a documented exception
- [ ] Direct dependencies are pinned or have version ranges that exclude known vulnerable versions

---

## Category 5: HTTP Security Headers

Verify these headers are set on all responses (check with browser DevTools > Network):

| Header | Expected value |
|--------|---------------|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` |
| `X-Content-Type-Options` | `nosniff` |
| `X-Frame-Options` | `DENY` or `SAMEORIGIN` |
| `Content-Security-Policy` | Restrictive policy blocking inline scripts |
| `Referrer-Policy` | `strict-origin-when-cross-origin` |

- [ ] CORS is configured with an explicit allowlist, not `*`
- [ ] Cookies have `HttpOnly`, `Secure`, and `SameSite` attributes

---

## Category 6: Data Validation on API Inputs

- [ ] Every API handler validates its input with a schema validator (Zod, Joi, etc.)
- [ ] Validation happens before any database or service call
- [ ] File upload endpoints have: size limits, type allowlist, no execution permissions
- [ ] Pagination parameters have maximum values (`take` cannot be arbitrarily large)
- [ ] Sort and filter parameters are validated against an allowlist of field names

---

## Category 7: Error Handling and Information Disclosure

- [ ] Production error responses do not include stack traces
- [ ] Production error responses do not include internal database error messages
- [ ] Generic error message is returned to the client; full detail is logged server-side only
- [ ] 404 responses do not reveal whether a resource exists vs the user lacks permission
  (both should return 404 -- use 403 only in contexts where the resource existence is already known)

---

## Category 8: Payment and Business Logic

- [ ] Subscription status for gated features is checked server-side, not client-side
- [ ] Stripe webhook signature verification is implemented and cannot be bypassed
- [ ] Stripe price IDs and amounts come from server-side config, not client inputs
- [ ] Users cannot access paid features by manipulating request parameters

---

## Reporting findings

For each finding, record:
```
SEVERITY: Critical | High | Medium | Low
CATEGORY: (from the categories above)
LOCATION: file:line
DESCRIPTION: What the vulnerability is
IMPACT: What an attacker could do
REMEDIATION: Specific fix
```

Severity guide:
- **Critical**: Can be exploited without authentication, affects data integrity or system availability
- **High**: Requires authentication to exploit, can access other users' data or escalate privilege
- **Medium**: Requires specific conditions, limited impact
- **Low**: Defense-in-depth improvement, no immediate risk

---

## Checklist

- [ ] Category 1 (Injection) -- complete
- [ ] Category 2 (Auth/Authz) -- complete
- [ ] Category 3 (Secrets) -- complete
- [ ] Category 4 (Dependencies) -- complete
- [ ] Category 5 (HTTP Headers) -- complete
- [ ] Category 6 (Input Validation) -- complete
- [ ] Category 7 (Error Handling) -- complete
- [ ] Category 8 (Payments) -- complete
- [ ] All Critical findings resolved before release
- [ ] All High findings resolved or have an accepted risk with documented exception
- [ ] Findings documented with severity and remediation
