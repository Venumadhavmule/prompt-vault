# WF08 -- Deployment Workflow

## Purpose

Step-by-step runbook for deploying any change to production safely. A deployment
without a runbook is a deployment without a plan. An unplanned deployment is where
production incidents happen.

---

## Pre-conditions

- All CI checks are passing on the branch being deployed
- PR has been approved and merged to main
- You have verified the same change in staging
- You have rollback access (ability to revert the deploy or run a rollback migration)

---

## Step 1: Pre-deployment checks

Run every item before initiating a deploy. No exceptions.

- [ ] All tests pass in CI on main
- [ ] `tsc --noEmit` passes locally
- [ ] No open P1 incidents currently active in production
- [ ] If DB migration is included: migration reviewed and classified as safe or expand phase
- [ ] If new env vars are required: they are already set in the production environment
- [ ] If a third-party integration changed: the new credentials are in production env
- [ ] Staging deploy was verified (smoke tested) with the same build artifact

---

## Step 2: Announce the deployment

For any change that affects user-visible behavior, notify the team before deploying:
```
[Deployment] Starting deploy of [version/description]
Includes: [brief list of changes]
Estimated completion: [time]
Rollback plan: [revert deploy / run rollback migration X]
```

---

## Step 3: Apply database migrations first

If the deploy includes schema changes:

```bash
# Run migrations against production BEFORE deploying new application code
npx prisma migrate deploy

# Verify migrations applied successfully
npx prisma migrate status
```

The migration must be backward compatible with the CURRENT application code.
If it is not, this is a breaking migration that was not planned with expand/contract.
Stop and follow WF05 instead.

---

## Step 4: Deploy the application

Execute the deployment via your deployment mechanism:

```bash
# Example: Railway
railway deploy

# Example: Vercel
vercel --prod

# Example: Docker/Kubernetes
docker build -t app:v1.2.3 .
docker push registry/app:v1.2.3
kubectl rollout restart deployment/app

# Example: manual process
git pull origin main
pnpm build
pm2 restart app
```

---

## Step 5: Verify deployment completed

Immediately after deploy, verify:

```bash
# Check the health endpoint
curl https://yourapp.com/api/health

# Expected response
{
  "status": "ok",
  "version": "1.2.3",
  "db": "connected"
}
```

If the health check fails, go to Step 7 (Rollback) immediately.

---

## Step 6: Post-deployment smoke tests

Within 5 minutes of deployment completing:

- [ ] Application is responding (health endpoint is green)
- [ ] Critical path works: authenticate, load main content, create/save core entity
- [ ] No spike in 5xx errors in your monitoring dashboard
- [ ] No spike in error logs
- [ ] DB migration (if any) is reflected in actual schema

If any smoke test fails, go to Step 7 immediately. Do not wait to see if it self-resolves.

---

## Step 7: Rollback procedure

If the deployment causes errors and cannot be quickly fixed with a follow-up deploy:

### Application rollback

```bash
# Redeploy the previous version

# Railway example: roll back to previous deployment in dashboard
# Vercel example: promote previous deployment in dashboard
# Kubernetes example:
kubectl rollout undo deployment/app

# PM2 example:
git checkout [previous-commit]
pnpm build
pm2 restart app
```

### Database rollback (if migration was applied)

If the migration was additive (added a nullable column or new table):
- Application rollback is sufficient. The old code ignores the new column.

If the migration changed or removed data:
- Restore from backup. Only valid if you took a backup before the migration.
- This is why you classify migrations before deploying.

**Document every rollback:**
```
Rollback executed: [date/time]
Reason: [what failed]
Steps taken: [what you did]
Status: [production is back to normal as of X]
Follow-up: [ticket to fix the root cause]
```

---

## Step 8: Post-deployment monitoring window

After every production deployment, watch the following for 15 minutes:

| Metric | Threshold for concern |
|--------|----------------------|
| Error rate (5xx) | Any increase above baseline |
| P95 response time | > 20% increase from baseline |
| Background job failure rate | Any increase |
| Database connection count | > 80% of pool capacity |
| Memory usage | Upward trend suggesting leak |

If any metric crosses the threshold: investigate. If you cannot determine why within
10 minutes, rollback and investigate with time on your side.

---

## Step 9: Announce completion

```
[Deployment Complete] [version/description]
Status: SUCCESS
Deployed at: [time]
Verified: [what was tested]
Monitoring: nominal
```

---

## Deployment checklist (summary card)

Pre-deploy:
- [ ] CI passing on main
- [ ] Staging verified
- [ ] New env vars set in production
- [ ] Migration reviewed and classified
- [ ] No active P1 incidents

Deploy:
- [ ] Team notified
- [ ] Migration applied (if applicable)
- [ ] Application deployed
- [ ] Health check passing

Post-deploy:
- [ ] Smoke tests passing
- [ ] No error rate increase
- [ ] No response time increase
- [ ] 15-minute monitoring window observed
- [ ] Team notified of completion
