# SKILL 4.1 -- How to Do a Security Review on Any Feature

## What this skill is

This skill is a practical security checklist and review process for use before any
feature ships. It translates the OWASP Top 10 from theory into specific code-level
patterns to look for. A security review is not a once-a-year activity -- it is a
gate that every feature passes through before it reaches production.

---

## When to use this skill

- Before approving any PR that adds new endpoints or changes auth/access logic
- Before shipping any feature that processes user input
- When doing a security audit pass on an existing codebase
- When a third-party security assessment is incoming
- When a new developer joins and is building their first feature

---

## Full Guide

### OWASP Top 10 as actionable code checks

---

#### 1. Injection (SQL, Command, LDAP)

**What to look for:**
```typescript
// VULNERABLE: string concatenation in queries
const user = await db.query(`SELECT * FROM users WHERE email = '${email}'`)

// SAFE: parameterized queries
const user = await db.query('SELECT * FROM users WHERE email = $1', [email])

// VULNERABLE: command injection
exec(`convert ${filename} output.jpg`)

// SAFE: no shell expansion
execFile('convert', [filename, 'output.jpg'])
```

Check:
- Every DB query uses parameterized queries or ORM (no string concatenation)
- No `eval()`, `Function()`, or dynamic code execution with user input
- No shell commands (`exec`, `spawn`) with user-controlled arguments
- File paths from user input are sanitized and sandboxed (`path.resolve` + check if inside allowed dir)

---

#### 2. Broken Authentication

Check:
- Access tokens are stored in-memory only (never localStorage, sessionStorage, or plain cookie)
- Refresh tokens are in httpOnly, Secure, SameSite=Strict cookies
- JWT signature is verified (not just decoded) before trusting claims
- Access token expiry is checked server-side on every request
- Password hashes use bcrypt with minimum 12 rounds
- Login attempts are rate-limited per account AND per IP
- Password reset tokens are hashed before DB storage, single-use, expire in ≤ 60 minutes

---

#### 3. Sensitive Data Exposure

Check:
- Passwords, tokens, and keys are NEVER in logs
- Passwords are NEVER returned in API responses
- Internal user IDs (auto-increment integers) are not exposed in URLs or responses (use UUIDs)
- Full credit card numbers, CVV, SSN, or health data are NEVER stored in your DB
- Audit logs do not contain sensitive fields (redact `password`, `token`, `secret`, `authorization`)
- HTTPS enforced in production (HSTS header set)
- Sensitive fields in the DB are either not stored or encrypted at rest if required by compliance

---

#### 4. Broken Access Control

This is the #1 vulnerability class. Every resource access must check ownership.

**The IDOR (Insecure Direct Object Reference) pattern:**
```typescript
// VULNERABLE: uses ID from URL without ownership check
app.get('/docs/:id', async (req, res) => {
  const doc = await db.doc.findById(req.params.id)
  return res.json(doc)
})

// SAFE: verifies resource belongs to authenticated user
app.get('/docs/:id', authMiddleware, async (req, res) => {
  const doc = await db.doc.findFirst({
    where: { id: req.params.id, userId: req.user.id } // ownership check
  })
  if (!doc) throw new NotFoundError('Document not found')
  return res.json(doc)
})
```

Check:
- Every endpoint that accesses a user-owned resource verifies ownership
- Admin routes have explicit role checks (not just auth)
- Users cannot access other tenants' data in multi-tenant apps
- Horizontal privilege escalation: can user A modify user B's resource?
- Vertical privilege escalation: can a regular user access admin endpoints?
- Pagination / list endpoints filter by the requesting user's scope

---

#### 5. Security Misconfiguration

Check:
- CORS origin is an explicit whitelist, NEVER `*` in production
- Error responses do not include stack traces, DB error messages, or file paths
- Security headers applied: `Content-Security-Policy`, `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, `Strict-Transport-Security`
- No default credentials in any service (DB, Redis, admin panels)
- Debug mode / verbose errors disabled in production
- `.env` files are in `.gitignore` and never committed
- Package manager lock files are committed and dependencies are not auto-upgraded in CI

---

#### 6. Vulnerable and Outdated Components

Check:
- `npm audit` runs in CI -- any high or critical vulnerabilities block deployment
- Dependencies are reviewed when upgrading major versions
- Docker base images are from official sources and pinned to specific digests
- Third-party packages inspected before adding (check download count, last publish, maintainers)

---

#### 7. Identification and Authentication Failures

Check:
- Session/token invalidation works on logout (verify refresh token is deleted from DB)
- Force logout all devices invalidates ALL refresh tokens for the user
- Email enumeration prevented: forgot-password always returns the same message
- Account lockout or CAPTCHA after repeated login failures
- Refresh token reuse detection: using a revoked token triggers family invalidation

---

#### 8. Software and Data Integrity Failures

Check:
- Webhook payloads are signature-verified before processing (Stripe uses `stripe-signature` header)
- File uploads validate MIME type and magic bytes (not just file extension)
- Dependency integrity checked: use `npm ci` not `npm install` in CI (uses lockfile)
- No `require()` or `import` of user-provided strings without extreme sandboxing

---

#### 9. Security Logging and Monitoring Failures

What must be logged (structured JSON, sent to your log aggregator):
- Every failed authentication attempt (timestamp, IP, email attempted -- not password)
- Every successful authentication (userId, IP, userAgent)
- Every privilege escalation attempt (user tried to access admin resource)
- Every password reset request and completion
- Every token reuse detection event
- Every Stripe webhook received and processed
- Every file upload (filename, MIME type, userId, outcome)

What must NEVER be logged:
- Passwords (even hashed)
- JWT tokens or refresh tokens
- Credit card numbers
- Session cookies
- Any header containing "authorization" or "cookie" (use redaction)

---

#### 10. SSRF (Server-Side Request Forgery)

Check:
- Does any endpoint accept a URL as input and fetch it server-side? (e.g., avatar URL, webhook URL)
- If yes: validate the URL against an allowlist of permitted domains
- Block internal IP ranges: `127.0.0.1`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `169.254.0.0/16`
- Use a DNS rebinding protection library for URL validation

---

### Dependency audit procedure

Run in CI and on-demand:
```bash
npm audit --audit-level=high    # fail on high+critical
```

Respond to findings:
- Critical/High: Fix before merge. No exceptions.
- Moderate: Create a ticket and fix within 2 weeks.
- Low: Review quarterly.

Dependency review on new packages:
- Check last publish date (> 1 year with no updates = risk)
- Check weekly downloads (low = low community vetting)
- Check GitHub issues for security reports
- Read the package's README and source for suspicious code

---

### Secret scanning

Before every commit (use pre-commit hook):
```bash
# Install: brew install trufflehog or pip install detect-secrets
trufflehog git file://. --since-commit HEAD --only-verified
```

If a secret is found in git history:
1. Immediately rotate the exposed credential (do not wait)
2. Use `git filter-repo` or BFG to remove from history
3. Force-push to all branches
4. Notify your security team

---

### The Security Checklist (25 items)

Before any feature ships:

**Input Handling**
- [ ] All user inputs validated via Zod schema at API boundary
- [ ] No raw SQL string concatenation anywhere in new code
- [ ] File uploads validate MIME type and magic bytes
- [ ] User-provided URLs are not fetched server-side without domain allowlist

**Authentication and Authorization**
- [ ] Auth middleware applied to all non-public endpoints
- [ ] Every resource access has an ownership check (IDOR prevention)
- [ ] Admin/privileged routes have explicit role checks
- [ ] JWT is verified (signature + expiry), not just decoded
- [ ] Refresh tokens are httpOnly cookie, stored hashed in DB

**Data Protection**
- [ ] No passwords, tokens, or sensitive PII in any log
- [ ] No sensitive data returned in API responses that is not needed
- [ ] No auto-increment IDs exposed in URLs or responses
- [ ] HTTPS enforced in production (HSTS header present)

**Configuration**
- [ ] CORS origin is explicit whitelist (no `*`)
- [ ] Security headers applied (CSP, X-Frame-Options, X-Content-Type-Options)
- [ ] Error responses do not expose stack traces or DB errors
- [ ] All secrets in env vars, .env committed to .gitignore

**Dependencies and Infrastructure**
- [ ] `npm audit` passes at high/critical level
- [ ] No new dependencies added without review
- [ ] Docker images from official sources, pinned versions

**Logging and Monitoring**
- [ ] Auth events (login success/failure, logout) are logged
- [ ] Suspicious access patterns produce observable log events
- [ ] Webhook signatures verified before processing

**Business Logic**
- [ ] Rate limiting applied to auth routes (login, register, forgot-password)
- [ ] Email enumeration prevented in forgot-password response
- [ ] Password reset tokens: single-use, expire in ≤ 60 minutes, hashed in DB

---

## What to avoid

DO NOT ship without running at least the top OWASP checks manually.

DO NOT treat security as "we'll add it later" -- it must be designed in from the start.

DO NOT use string concatenation for any query -- use parameterized queries always.

DO NOT allow `*` CORS in production for any non-public anonymous API.

DO NOT log anything that could be a secret or token.

DO NOT ship a webhook endpoint without verifying the provider's signature.
