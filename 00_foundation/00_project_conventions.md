# Project Conventions -- Living Document

> This file is the single source of truth for all project-level conventions.
> Every contributor (human or AI agent) MUST follow these rules without exception.

---

## 1. Monorepo Folder Structure

```
root/
  apps/
    web/                  # Next.js frontend (App Router)
    api/                  # Backend API server (Hono / Express / Fastify)
    worker/               # Background job processor (BullMQ)
    admin/                # Admin panel (optional)
  packages/
    shared/               # Shared types, utils, constants across apps
    ui/                   # Shared UI component library (shadcn/ui based)
    db/                   # Prisma schema, migrations, seed, repositories
    config/               # Shared ESLint, TSConfig, Tailwind configs
    email/                # Email templates (React Email or MJML)
    validation/           # Shared Zod schemas for cross-app validation
  infra/
    docker/               # Dockerfiles, docker-compose files
    terraform/            # IaC for cloud resources (optional)
    scripts/              # Deploy scripts, DB backup scripts, seed scripts
  docs/
    architecture/         # ADRs (Architecture Decision Records)
    api/                  # OpenAPI specs, Postman collections
    runbooks/             # Operational runbooks for incidents
  .github/
    workflows/            # GitHub Actions CI/CD
    PULL_REQUEST_TEMPLATE.md
    ISSUE_TEMPLATE/
  .env.example            # All env vars with descriptions
  turbo.json              # Turborepo pipeline config
  package.json            # Root workspace config
```

---

## 2. File Naming Rules

| Context              | Convention   | Example                          |
|----------------------|--------------|----------------------------------|
| Files (general)      | kebab-case   | `user-service.ts`                |
| React components     | PascalCase   | `UserProfile.tsx`                |
| Functions/variables  | camelCase    | `getUserById`, `isActive`        |
| Constants            | SCREAMING_SNAKE | `MAX_RETRY_COUNT`             |
| Types/Interfaces     | PascalCase   | `UserPayload`, `CreateUserInput` |
| Enums                | PascalCase   | `SubscriptionStatus`             |
| Enum members         | SCREAMING_SNAKE | `ACTIVE`, `PAST_DUE`          |
| Database tables      | snake_case plural | `users`, `refresh_tokens`   |
| Database columns     | snake_case   | `created_at`, `is_verified`      |
| Environment vars     | SCREAMING_SNAKE | `DATABASE_URL`, `JWT_SECRET`  |
| Test files           | Same as source + `.test` | `user-service.test.ts`|
| Migration files      | Timestamp prefix | `20240315_create_users.sql`  |

---

## 3. Git Commit Message Format (Conventional Commits)

```
<type>[optional scope][optional !]: <summary in imperative mood>

<description: WHAT changed and WHY>
```

### Types
- `feat`: New feature (MINOR version bump)
- `fix`: Bug patch (PATCH version bump)
- `refactor`: Code restructure, no behavior change
- `chore`: Tooling, deps, config changes
- `update`: Enhance existing feature
- `docs`: Documentation only
- `build`: Build system or CI changes
- `test`: Adding or fixing tests
- `perf`: Performance improvement

### Rules
- Summary line: imperative mood, present tense, start with verb
- No period at end of summary
- Body: one blank line after summary, explain WHAT and WHY
- Breaking changes: append `!` after type/scope, explain in body
- Max summary length: 72 characters

### Examples
```
feat(auth): Add refresh token rotation with reuse detection

Implement automatic refresh token rotation on each token refresh call.
Store token family ID to detect reuse. When reuse is detected, revoke
all tokens in the family and force re-authentication. Prevents token
theft replay attacks.

fix(payments): Correct proration calculation for mid-cycle upgrades

The proration was using the subscription start date instead of the
current billing period start. This caused incorrect charge amounts
when upgrading mid-cycle. Now uses Stripe's proration_date parameter
set to the current timestamp.
```

---

## 4. Branch Naming

| Purpose        | Pattern                    | Example                              |
|----------------|----------------------------|--------------------------------------|
| Feature        | `feature/<ticket>-<slug>`  | `feature/PV-42-add-stripe-webhooks`  |
| Bug fix        | `fix/<ticket>-<slug>`      | `fix/PV-88-token-expiry-race`        |
| Hotfix         | `hotfix/<slug>`            | `hotfix/payment-double-charge`       |
| Release        | `release/<version>`        | `release/1.3.0`                      |
| Chore/Tooling  | `chore/<slug>`             | `chore/upgrade-prisma-5`             |
| Experiment     | `spike/<slug>`             | `spike/websocket-scaling`            |

---

## 5. Pull Request Checklist

Every PR MUST satisfy all items before merge:

- [ ] TypeScript compiles with zero errors (`tsc --noEmit`)
- [ ] All tests pass (`pnpm test`)
- [ ] Lint passes with zero warnings (`pnpm lint`)
- [ ] New DB changes include migration file (forward + rollback)
- [ ] `.env.example` updated if new env vars added
- [ ] Zod schemas updated for new/changed request/response shapes
- [ ] No `console.log` left in production code (use structured logger)
- [ ] No `any` type unless absolutely unavoidable (with `// eslint-disable` + comment explaining why)
- [ ] Error cases handled (no unhandled promise rejections)
- [ ] API changes documented (OpenAPI spec updated)
- [ ] CHANGELOG updated for user-facing changes
- [ ] Screenshots/recording attached for UI changes
- [ ] PR description explains WHAT and WHY (not just HOW)

---

## 6. Code Review Standards (6 Dimensions)

| Dimension        | What to check                                                              |
|------------------|----------------------------------------------------------------------------|
| **Correctness**  | Does it do what the ticket says? Edge cases handled? Off-by-one errors?    |
| **Security**     | Input validated? Auth checked? Injection-safe? Secrets in env vars only?   |
| **Performance**  | N+1 queries? Missing indexes? Unbounded queries? Large payloads?           |
| **Maintainability** | Clear naming? Single responsibility? Easy to modify later?              |
| **Test Coverage** | Unit tests for logic? Integration tests for routes? Edge cases tested?    |
| **Developer Experience** | Types helpful? Errors descriptive? Logs useful for debugging?      |

---

## 7. Code Organization Within Files

```typescript
// 1. External imports (node_modules)
import { z } from 'zod';
import { Hono } from 'hono';

// 2. Internal imports (project packages)
import { AppError } from '@packages/shared/errors';
import { env } from '@packages/config/env';

// 3. Relative imports (same package)
import { UserRepository } from './user-repository';
import type { CreateUserInput } from './types';

// 4. Types and interfaces
interface ServiceDeps {
  userRepo: UserRepository;
  emailService: EmailService;
}

// 5. Constants
const MAX_LOGIN_ATTEMPTS = 5;

// 6. Main class/function exports
export class UserService {
  // constructor first
  // public methods next (alphabetical)
  // private methods last (alphabetical)
}

// 7. Helper functions (not exported, at bottom)
function formatUserResponse(user: User): UserResponse {
  // ...
}
```

---

## 8. Environment Setup Steps

```bash
# Prerequisites
node --version    # Must be >= 20.0.0 (use fnm or nvm)
pnpm --version    # Must be >= 9.0.0

# Clone and install
git clone <repo-url>
cd <project>
pnpm install

# Environment
cp .env.example .env
# Fill in required values (see .env.example for descriptions)

# Database
docker compose up postgres redis -d
pnpm db:migrate      # Run migrations
pnpm db:seed         # Seed development data

# Development
pnpm dev             # Starts all apps in parallel (via Turborepo)
pnpm dev --filter=api   # Start only the API
pnpm dev --filter=web   # Start only the frontend

# Testing
pnpm test            # Run all tests
pnpm test:unit       # Unit tests only
pnpm test:integration # Integration tests (needs running DB)
pnpm test:e2e        # E2E tests (needs running app)

# Linting and formatting
pnpm lint            # ESLint check
pnpm lint:fix        # ESLint auto-fix
pnpm format          # Prettier format
pnpm typecheck       # TypeScript strict check
```

---

## 9. Dependency Management Rules

- Pin exact versions in `package.json` (no `^` or `~`)
- Run `pnpm audit` weekly; fix critical/high vulnerabilities within 24 hours
- Major version upgrades go through a dedicated PR with migration notes
- Never install a package without checking: bundle size, maintenance status, last publish date, open security advisories
- Prefer packages with TypeScript types built-in over `@types/*` when available

---

## 10. Logging Rules

- NEVER use `console.log`, `console.error`, `console.warn` in production code
- Use the structured logger from `@packages/shared/logger`
- Every log line MUST include: `requestId`, `level`, `message`, `timestamp`
- NEVER log: passwords, tokens, API keys, credit card numbers, PII beyond userId
- Log levels: `debug` (local only), `info` (normal operations), `warn` (recoverable issues), `error` (failures needing attention), `fatal` (app cannot continue)
