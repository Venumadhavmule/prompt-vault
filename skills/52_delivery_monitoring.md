# SKILL 5.3 -- How to Monitor and Observe a Running System

## What this skill is

This skill defines how to set up and use observability in production -- specifically
logs, metrics, and traces -- to understand system health, detect problems before
users report them, and investigate incidents efficiently. You cannot manage what you
cannot observe. This skill ensures you always know what your system is doing and can
explain any incident with precision.

---

## When to use this skill

- When setting up monitoring for a new service prior to going live
- When investigating a production incident
- When the team is receiving noisy alerts that are not useful
- When a user-reported incident cannot be reproduced from available data
- When setting up structured logging for the first time in a project

---

## Full Guide

### The three pillars of observability

**Logs:** What happened, in sequence. The narrative of events.
- Structured (JSON) in production
- Searchable and filterable
- Used for: debugging specific requests, tracing user actions, audit trails

**Metrics:** What is happening, quantified over time. Numbers with context.
- Time-series data (latency, request rate, error rate, memory)
- Used for: dashboards, health checks, capacity planning, alerts

**Traces:** HOW a request moved through the system. The path and timing.
- Spans across services, DB calls, and external calls
- Used for: latency attribution, finding which step is slow in a complex flow

---

### The four golden signals

These four metrics, applied to any service, indicate whether the service is healthy:

| Signal | What it measures | Alert threshold |
|---|---|---|
| Latency | How long requests take (p50, p95, p99) | Alert when p95 > 500ms (set per service) |
| Traffic | Request rate (req/s) | Alert on sudden drops (not just spikes) |
| Errors | Error rate (4xx, 5xx per total requests) | Alert when 5xx error rate > 1% |
| Saturation | How "full" the service is (CPU, memory, DB connections, queue depth) | Alert at 80% capacity |

These four should be on every service dashboard. If you can only set up four metrics,
make it these four.

---

### How to write useful structured logs

**Required fields on every log entry:**
```json
{
  "timestamp": "2024-11-15T14:22:33.123Z",
  "level": "info",
  "message": "User login successful",
  "service": "api",
  "version": "1.3.2",
  "environment": "production",
  "requestId": "req_abc123",
  "userId": "user_def456",
  "method": "POST",
  "path": "/api/v1/auth/login",
  "statusCode": 200,
  "durationMs": 45
}
```

**Log levels (use them consistently):**
- `debug`: Verbose details for development troubleshooting. Never in production.
- `info`: Normal operational events (user logged in, job completed, payment processed)
- `warn`: Unexpected but handled situations (rate limit hit, invalid token, retry attempt)
- `error`: Failures that require investigation (unhandled exception, DB error, Stripe failure)
- `fatal`: The service cannot continue running (startup failure, critical dependency unavailable)

**What to log:**
- Every request: method, path, status, duration, requestId
- Every auth event: login, logout, token refresh, password change
- Every payment event: checkout started, payment intent created, webhook received/processed
- Every job: started, completed, failed (with job type and ID)
- Every external service call: name of service, method, duration, outcome

**What to NEVER log:**
- Passwords (even hashed)
- JWT tokens or refresh tokens
- Cookie values
- Credit card numbers, CVV, expiry
- SSN, health data, or any field that is PII under GDPR
- Authorization headers verbatim
- Request body fields named: password, token, secret, credentials, authorization, card, cvv

Configure automatic redaction in your logger for these field names.

---

### Setting up alerts that are useful

Alert on symptoms, not causes.

**Good alert:**
"p95 response time for the API service exceeded 800ms for 5 consecutive minutes"
This is what users experience. It triggers investigation.

**Bad alert:**
"CPU usage exceeded 70%"
CPU at 70% means nothing unless it affects the four golden signals.

**Alert rules:**
- Alert must have an owner -- who gets paged?
- Alert must have a runbook link -- what does the responder do?
- Alert must be actionable -- if you cannot act on it, do not alert on it
- Alert must have a window -- not every spike, but sustained degradation
- Alert must have a threshold -- pre-set, not ad-hoc

**Alert severity tiers:**
| Tier | Response | Example |
|---|---|---|
| P1 Critical | Page immediately (24/7) | Checkout is down, payments failing, 100% error rate |
| P2 High | Notify on-call (business hours) | Error rate > 5%, p99 latency > 3s |
| P3 Medium | Ticket created, fix in sprint | p95 latency slightly elevated, disk at 75% |
| P4 Low | Informational dashboard | Unusual pattern, non-impacting anomaly |

---

### Investigating an incident

When an alert fires:

```
1. Acknowledge the alert -- prevent duplicate paging

2. Assess impact
   - How many users are affected?
   - Is revenue/core functionality impacted?
   - What is the error rate?

3. Find the starting point
   - When did it start? (Check the metrics timeline)
   - What deployed at or before that time? (Check deployment log)

4. Correlate logs
   - Filter by time window of the incident
   - Filter by high error rate or specific error code
   - Find the first occurrence of the error
   - Note the requestId, trace the specific request

5. Form and test hypothesis (use Skill 3.6: debugging)

6. Fix or mitigate
   - Mitigation first if users are impacted (rollback, feature flag off, circuit breaker)
   - Then investigate root cause

7. Declare resolved when error rate returns to baseline

8. Write post-incident review within 48 hours
```

---

### Post-incident review format

```markdown
# Post-Incident Review: [Incident Title]

**Date**: [date]
**Severity**: P1 / P2 / P3
**Duration**: [start time] to [end time] ([total duration])
**Impact**: [how many users affected, what functionality was down]

## Timeline

| Time | Event |
|------|-------|
| 14:22 | Error rate alert fires |
| 14:24 | On-call engineer acknowledges |
| 14:31 | Deploy identified as likely cause |
| 14:35 | Rollback initiated |
| 14:38 | Error rate returns to baseline |
| 14:40 | Incident declared resolved |

## Root Cause

[1-2 paragraphs: what exactly caused the incident]

## Why It Wasn't Caught Earlier

[What monitoring, test, or process failed to catch this before production]

## What Went Well

[What parts of the response worked correctly]

## What Went Poorly

[What slowed down detection, diagnosis, or resolution]

## Action Items

| Action | Owner | Due Date |
|--------|-------|----------|
| Add integration test for [failing scenario] | [name] | [date] |
| Add alert for [missed signal] | [name] | [date] |
| Update runbook with [new procedure] | [name] | [date] |
```

---

### Monitoring setup checklist before production launch

```
Infrastructure:
[ ] Health check endpoint at /health (returns DB + Redis status)
[ ] Metrics endpoint at /metrics (Prometheus format)
[ ] Sentry (or equivalent) configured and capturing unhandled errors
[ ] Structured JSON logging in production, logs shipped to aggregator (Datadog, Loki, etc.)
[ ] Log retention policy set (at least 30 days for operational, 1 year for audit)

Alerts:
[ ] P1 alert: error rate > 5% for any 5-minute window
[ ] P1 alert: health check fails for 2+ consecutive minutes
[ ] P2 alert: p95 latency > 500ms for 10+ minutes
[ ] P2 alert: DB connection pool > 80% for 5+ minutes
[ ] P3 alert: disk space > 80%
[ ] Alert notifications configured (PagerDuty, OpsGenie, Slack)

Dashboards:
[ ] Four golden signals dashboard (latency, traffic, errors, saturation)
[ ] DB performance dashboard (query time, connection pool, slow queries)
[ ] Business metrics dashboard (signups, active users, revenue -- to detect functional regressions)
[ ] External service health (Stripe status, email delivery rate)
```

---

## What to avoid

DO NOT log PII, passwords, tokens, or secrets in any log destination.

DO NOT alert on causes (CPU %) instead of symptoms (p95 latency, error rate).

DO NOT create alerts without a runbook. An alert that fires tells you nothing if responders don't know what to do.

DO NOT skip the post-incident review, even for P3 incidents. They repeat without reviews.

DO NOT use unstructured log formats in production. You cannot search "user_id:abc123" in plaintext logs.

DO NOT set up monitoring only AFTER going live. Set it up before the first production deploy.
