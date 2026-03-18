# SKILL 5.4 -- How to Write Documentation That Gets Read

## What this skill is

This skill covers how to write documentation that is actually useful -- the kind
developers read, not the kind they file away. Most documentation fails for one of
three reasons: it is too long, it becomes outdated immediately, or it describes what
the code does instead of why. This skill produces documentation that developers
trust because it is accurate, scannable, and explains the decisions behind the code.

---

## When to use this skill

- When setting up a new project or service
- When writing a README for a repo
- When documenting a new API endpoint
- When writing a comment that explains unusual logic
- When writing an Architecture Decision Record
- When onboarding a new developer to an existing codebase

---

## Full Guide

### The documentation types

| Type | When to write | Lives in |
|---|---|---|
| README | When creating a new service or package | `/README.md` |
| API docs | When adding or changing an endpoint | `/docs/api/` or OpenAPI annotations |
| Architecture Decision Record (ADR) | When making a significant technical decision | `/docs/architecture/` |
| Runbook | When deploying a service to production | `/docs/runbooks/` |
| Inline comments | When code logic is non-obvious | Next to the code |
| CHANGELOG | When releasing changes | `/CHANGELOG.md` |
| Technical design doc | Before building something complex | `/docs/design/` |

---

### Step 1: How to write a README that works in under 10 minutes

A developer landing on your README cold needs to be productive within 10 minutes.
If they cannot, the README has failed.

**Required sections (in this order):**

```markdown
# [Service / Project Name]

[One sentence: what this is and what it does]

## Requirements

[Exact versions: Node 20.x, pnpm 8.x, Docker, etc.]

## Setup

[Commands to go from zero to running locally. Every command.
Do not say "configure your environment" -- write exactly what to run.]

\`\`\`bash
cp .env.example .env
pnpm install
pnpm db:migrate
pnpm dev
\`\`\`

## Available Commands

| Command | What it does |
|---------|-------------|
| `pnpm dev` | Start dev server with hot reload |
| `pnpm build` | Build for production |
| `pnpm test` | Run all tests |
| `pnpm test:unit` | Run unit tests only |
| `pnpm db:migrate` | Apply pending migrations |
| `pnpm db:studio` | Open Prisma Studio |

## Architecture

[2-3 paragraphs: how is this structured? What are the main folders and what lives where?]

## Key Concepts

[Any project-specific concepts a developer needs to know before making changes]

## Configuration

[Table of all env vars, their type, description, and whether they are required or optional]

## API Reference

[Link to generated docs or brief endpoint summary]

## Tests

[How to run tests, what kinds of tests exist, where they live]

## Deployment

[Link to runbook or brief deploy instructions]
```

**Validation:** Have a developer who has never seen this project do a local setup
using only the README. Time them. If it takes more than 10 minutes, update the README.

---

### Step 2: Inline comments that add value

**Comment the WHY, not the WHAT.**

The code already shows WHAT it does. Comments must explain WHY the code does it
this way, especially when the reason is not obvious from the implementation.

**Bad comment (explains what, which is visible):**
```typescript
// Loop through users
for (const user of users) {
```

**Good comment (explains why):**
```typescript
// Prisma does not support bulk upsert natively, so we fall back to
// individual upserts in a sequential loop. This is intentionally slow
// but this runs only in dev seeding, never in the hot path.
for (const user of users) {
```

**When to write a comment:**
- When a solution looks wrong but is correct for a specific reason
- When the code references a specific issue, bug fix, or ticket ("See JIRA-2341")
- When a business rule is being enforced and the rule itself is not obvious from context
- When choosing between two options and explaining why this one was chosen

**When NOT to write a comment:**
- On every function (write good function names instead)
- On every line (your code is too complex if it needs per-line comments)
- On obvious operations

---

### Step 3: API documentation format

Every API endpoint must have documentation a frontend developer can build against
without asking questions. Use this format:

```markdown
## POST /api/v1/auth/login

Authenticate a user and return access and refresh tokens.

**Auth required**: No
**Rate limit**: 10 per minute per IP

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| email | string | Yes | User's email address |
| password | string | Yes | User's password (min 8 characters) |

### Success Response (200)

\`\`\`json
{
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIs...",
    "expires_in": 900,
    "user": {
      "id": "usr_abc123",
      "email": "user@example.com",
      "first_name": "Alice"
    }
  }
}
\`\`\`

The refresh token is set as an httpOnly cookie (not in the response body).

### Error Responses

| Status | Code | When |
|--------|------|------|
| 400 | VALIDATION_ERROR | Email or password missing/malformed |
| 401 | INVALID_CREDENTIALS | Email not found or password incorrect |
| 403 | EMAIL_NOT_VERIFIED | Account exists but email not verified |
| 429 | RATE_LIMIT_EXCEEDED | Too many attempts |
```

---

### Step 4: Technical design document format

Before building something complex (a new core module, an integration, an async
processing system), write a design doc first. It surfaces problems before code is written.

```markdown
# Technical Design: [Feature Name]

**Author**: [name]
**Date**: [date]
**Status**: Draft | Under Review | Approved | Implemented

## Problem

[What problem are we solving? Why does it need to be solved now?]

## Goals

- [What this solution must achieve]

## Non-Goals

- [What this solution is explicitly NOT doing]

## Proposed Solution

[Description of the approach: how does it work at a high level?]

### Architecture

[Diagram or description of the components and how they interact]

### Data Model

[New tables, modified tables, schema changes]

### API Interface

[New endpoints or changes to existing endpoints]

### Sequence Diagram

[Step-by-step: who calls what in what order]

## Alternatives Considered

### Alternative 1: [Name]
- Pros: [...]
- Cons: [...]
- Rejected because: [...]

## Open Questions

- [ ] [Question that needs to be resolved before implementation]

## Implementation Plan

[High-level phases or milestones]

## Success Metrics

[How will we know this worked? What measurements confirm success?]
```

---

### Step 5: CHANGELOG format

The CHANGELOG records every user-facing change. It is for users and developers,
not for git commits (which are for yourself).

```markdown
# Changelog

## [Unreleased]

## [1.4.0] -- 2024-11-15

### Added
- Password reset via email
- Google OAuth login
- CSV export for invoices

### Changed
- Access token expiry reduced from 1 hour to 15 minutes for security

### Fixed
- Subscription did not reactivate after failed payment was resolved
- Email was case-sensitive during login (now normalized to lowercase)

### Security
- Refresh tokens now rotated on every use with reuse detection

## [1.3.2] -- 2024-10-28

### Fixed
- Race condition in token refresh endpoint
```

---

### Step 6: Keeping documentation up to date

Documentation decays the moment it stops being part of the "definition of done."

Rules:
- Add documentation changes to the same PR as the code changes
- Every PR checklist includes: "README/docs updated if applicable"
- No feature is "done" until the API docs reflect the current endpoint contract
- Every migration must include a CHANGELOG entry
- ADRs are never deleted -- their status is updated to "Superseded by ADR-N"

**Signs documentation is failing:**
- Developers ask questions that the README should answer
- API docs and actual API behavior disagree
- A new team member cannot set up locally from the README alone
- ADRs say "Proposed" but the feature shipped months ago

---

## What to avoid

DO NOT write documentation as an afterthought after the feature ships.

DO NOT write what the code does -- write why decisions were made.

DO NOT commit documentation that you know is already out of date.

DO NOT write a 10,000-word README. Scannable and short always beats comprehensive and unread.

DO NOT skip the CHANGELOG for releases that change user-facing behavior.

DO NOT write comments on every line -- only where the reader would otherwise be confused.

---

## Checklist

Before closing any feature as complete:

- [ ] README updated if setup, commands, or architecture changed
- [ ] .env.example updated with any new env vars (with description and example value)
- [ ] API documentation updated for every new or changed endpoint
- [ ] Inline comments added where logic is non-obvious
- [ ] CHANGELOG entry added for user-facing changes
- [ ] ADR written and stored in `docs/architecture/` for any significant technical decision
- [ ] New service or module has a runbook in `docs/runbooks/`
- [ ] A new developer could set up locally using the README without asking questions
- [ ] Documentation is in the same PR as the code changes (not a follow-up)
