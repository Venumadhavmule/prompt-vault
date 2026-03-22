# SKILL 5.2 -- How to Deploy Software Safely

## What this skill is

This skill defines the process for deploying any service to production without
causing downtime, data loss, or user impact. Most deployment incidents are caused
not by the code change itself but by the deployment process: a migration that locks
a table, a configuration change that was missing from staging, or a broken rollback
plan. This skill makes each deployment deliberate, verified, and reversible.

---

## When to use this skill

- Before every deployment to a staging or production environment
- When planning a release that includes a database migration
- When deploying a feature with infrastructure changes
- When you are on-call and performing an emergency rollback

---

## Full Guide

### The pre-deploy checklist

Do not begin deployment without verifying every item:

```
Pre-deploy verification:
[ ] All CI checks pass (lint, type-check, unit tests, integration tests)
[ ] No open [must fix] review comments on the PR
[ ] Migration written and tested (if schema changed)
[ ] Rollback migration written alongside the forward migration
[ ] .env variables added to all target environments (staging, prod secrets manager)
[ ] .env.example is up to date
[ ] Third-party service configuration updated (e.g., new Stripe webhook event added to dashboard)
[ ] Feature flags set correctly if using gradual rollout
[ ] Stale code cleanup completed (old code paths that depended on old schema removed if applicable)
[ ] Load test or smoke test on staging (for changes to critical paths)
[ ] Database backup verified and newer than 24 hours
[ ] On-call person identified and reachable during the deploy
[ ] Rollback plan documented (what command, what migration to run to reverse)
[ ] Stakeholders notified if there is any user-visible change
```

---

### Deploy database migrations safely

Migrations are the highest-risk part of any deployment. Always separate migration
deployment from application code deployment (expand/contract pattern):

**Phase 1: Expand (backward compatible changes)**
- Deploy migration that adds new columns as nullable
- The old code continues to work (it ignores the new column)

**Phase 2: Application update**
- Deploy new application code that reads AND WRITES the new column
- Old code was writing nulls; new code writes values

**Phase 3: Backfill (if needed)**
- Run a background backfill to populate the new column for existing rows

**Phase 4: Contract (make it required)**
- Add NOT NULL constraint after all rows have values
- Drop old columns after all code references are removed

**Never do in one deployment:**
- Add NOT NULL column + deploy new code atomically (migration can fail on large tables)
- Drop a column that application code still reads
- Rename a column without a multi-phase approach

**Order for a safe migration deploy:**
1. Take DB backup
2. Deploy migration to staging → verify
3. Deploy migration to production (off-peak hours preferred)
4. Verify migration completed: check row counts, spot-check data
5. Then deploy application code

---

### Deployment strategies

**Rolling deploy:**
- Gradually replace old instances with new ones
- At any moment, some instances run the old code, some run the new
- Requires backward compatibility for the transition period
- Works well for stateless services
- Use for: routine feature deploys

**Blue/Green deploy:**
- Run two identical environments (blue = current, green = new)
- Switch traffic to green only after full health check
- Blue stays up for instant rollback
- Use for: high-risk changes where immediate rollback is critical
- Cost: requires double infrastructure during the transition

**Canary deploy:**
- Route a small % of traffic (1-5%) to the new version
- Monitor error rates, latency, and business metrics
- Gradually increase percentage if metrics are healthy
- Instantly roll back to 0% if anomalies detected
- Use for: large-scale changes or when you want safety with gradual validation

---

### Environment variable management

Never store environment variable values in code or configuration files that are committed.

Best practices:
- Development: `.env` file (in `.gitignore`), all values in `.env.example` with placeholder
- Staging/Production: secrets manager (AWS Secrets Manager, Doppler, Vault, GitHub Actions secrets)
- Per-environment overrides: separate secret sets per environment (staging DB ≠ prod DB)

Before deploying:
- Confirm every new env var is set in the target environment's secret store
- Confirm values are environment-appropriate (staging Stripe keys, not prod keys, in staging)

---

### Verifying a deployment succeeded

Run these checks immediately after every deployment:

**1. Health check:**
```bash
curl https://api.yourdomain.com/health
# Expected: { "status": "ok", "db": "connected", "redis": "connected", "version": "1.3.4" }
```

**2. Smoke tests:**
- Hit the 5 most critical API endpoints with valid data and verify expected responses
- Log in as a test user, perform the core user action
- Check that Stripe webhooks are routing correctly (if applicable)

**3. Error rate monitoring:**
After deploy, watch error rate in your monitoring dashboard for 10-15 minutes.
- If error rate > 1% above baseline → consider rollback
- If p95 latency doubles → investigate before proceeding

**4. Log inspection:**
Scan the live structured logs for `ERROR` or `WARN` that did not exist before the deploy.

---

### Rollback trigger conditions

Rollback immediately without additional diagnosis if any of these occur:

- Error rate increases by more than 2x within 5 minutes of deploy
- p95 response time increases by more than 3x
- DB connection pool is exhausting
- Critical user journey is broken (login, payment, core feature)
- Revenue-affecting bug confirmed (payment not processing, subscriptions not activating)

Rollback candidates (investigate before deciding):
- Isolated reported errors that affect < 1% of users
- A non-critical feature that is broken
- Performance regression not affecting the critical path

---

### How to rollback

**Application code rollback:**
```bash
# Kubernetes
kubectl rollout undo deployment/api

# Docker Compose / server deploy
git checkout <previous-stable-tag>
./scripts/deploy.sh

# Vercel / Netlify: use the dashboard "Rollback to previous deployment"
```

**Database migration rollback:**
```bash
# Prisma
npx prisma migrate resolve --rolled-back <migration-name>
# Then manually run your rollback SQL

# Or: run the rollback migration file directly
psql $DATABASE_URL < migrations/20240315_add_avatar_url.rollback.sql
```

Always test the rollback migration on staging before running it in production.

---

### Writing a runbook

Every production service needs a runbook. Store in `docs/runbooks/[service].md`.

**Runbook template:**
```markdown
# [Service Name] Runbook

## Purpose
[One sentence: what this service does]

## Architecture
[Brief: what it depends on (DB, Redis, external APIs)]

## Deployment

### Deploy
[Commands to deploy]

### Verify
[Commands to verify deploy succeeded — health check URL, smoke test description]

### Rollback
[Commands to rollback -- must be runnable in under 5 minutes]

## Common Issues

### Issue: [name]
**Symptoms**: [what you see]
**Root cause**: [why it happens]
**Resolution**: [exact steps]

## Monitoring
- Dashboard: [link]
- Alerts: [what alerts are set up and what they mean]
- Logs: [where to find them, what to search for]

## Contacts
- On-call: [how to reach]
- Escalation: [who to escalate to]
```

---

### Post-deploy checklist

```
After deployment, within 30 minutes:
[ ] Health check returns 200
[ ] All smoke tests pass
[ ] Error rate is within normal baseline for 10+ minutes
[ ] p95 latency is within normal baseline
[ ] No new ERROR or WARN log patterns appeared
[ ] Monitoring dashboard is not showing alerts
[ ] Test the specific feature that was deployed manually as a user
[ ] Notify team that deploy is complete and verified
[ ] Update deployment log or ticket to "deployed"
```

---

## What to avoid

DO NOT deploy after 3pm on Fridays (or before a long weekend).

DO NOT deploy without a rollback plan.

DO NOT deploy a database migration and new application code simultaneously in a single step for large or complex changes.

DO NOT proceed if CI is failing. Fix the failure first.

DO NOT deploy without verifying env vars are set in the target environment.

DO NOT skip the post-deploy monitoring window. Incidents often surface 5-10 minutes after deploy, not instantly.

---

## Checklist

Before every production deployment:

- [ ] All CI checks pass
- [ ] Pre-deploy checklist complete
- [ ] Migration strategy decided (separate or combined based on risk level)
- [ ] Rollback plan written and tested
- [ ] New env vars confirmed in target environment
- [ ] Health check endpoint defined and working
- [ ] On-call engineer aware of deployment
- [ ] Deploy executed in staging first and verified
- [ ] Production deploy executed
- [ ] Health check returns 200 within 2 minutes
- [ ] Smoke tests pass
- [ ] Error rate and latency monitored for 15 minutes
- [ ] Post-deploy checklist complete
- [ ] Runbook updated if any operational procedure changed
