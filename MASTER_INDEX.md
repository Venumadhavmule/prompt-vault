# MASTER INDEX -- Prompt Vault Skill and Workflow Library

This index covers every skill file and workflow in this library.
Use the lookup tables to find the right file for any situation.

---

## Quick Navigation

- [Foundation Files](#foundation-files)
- [Skills: Category 1 -- Mindset](#skills-category-1----mindset)
- [Skills: Category 2 -- Planning](#skills-category-2----planning)
- [Skills: Category 3 -- Execution](#skills-category-3----execution)
- [Skills: Category 4 -- Quality](#skills-category-4----quality)
- [Skills: Category 5 -- Delivery](#skills-category-5----delivery)
- [Skills: Category 6 -- AI Agent Skills](#skills-category-6----ai-agent-skills)
- [Workflows](#workflows)
- [Situation Lookup Table](#situation-lookup-table)
- [AI Agent Quick-Start Guide](#ai-agent-quick-start-guide)

---

## Foundation Files

| File | What it covers |
|------|----------------|
| `skills/00_project_conventions.md` | Monorepo structure, naming conventions, git format, PR checklist, code review standards |
| `skills/00_tech_stack_decision_matrix.md` | Next.js vs Remix vs Vite, PostgreSQL vs MongoDB, REST vs tRPC vs GraphQL, Prisma vs Drizzle, comparison tables |

---

## Skills: Category 1 -- Mindset

| File | Skill | One-line summary |
|------|-------|-----------------|
| `skills/10_mindset_read_task.md` | 1.1 Read and Understand Any Task | How to extract the exact problem before writing code |
| `skills/11_mindset_break_problems.md` | 1.2 Break Any Problem Into Subtasks | Decompose features into independently completable units |
| `skills/12_mindset_technical_decisions.md` | 1.3 Make Technical Decisions | Trade-off tables, ADR format, build vs buy, 2-year regret filter |

---

## Skills: Category 2 -- Planning

| File | Skill | One-line summary |
|------|-------|-----------------|
| `skills/20_planning_feature.md` | 2.1 Plan a New Feature End to End | User story → acceptance criteria → data model → API contract → test plan |
| `skills/21_planning_database.md` | 2.2 Plan a Database Schema | Entity design, relationships, indexes, migration safety, naming conventions |
| `skills/22_planning_api.md` | 2.3 Plan an API | RESTful naming, endpoint contracts, versioning, error format, pagination |
| `skills/23_planning_auth.md` | 2.4 Plan Authentication | Auth method decision tree, token lifecycle, RBAC vs ABAC, compliance |

---

## Skills: Category 3 -- Execution

| File | Skill | One-line summary |
|------|-------|-----------------|
| `skills/30_execution_implement_feature.md` | 3.1 Implement Any Feature | Mandatory layer order: types → schema → migration → repo → service → API → UI → tests |
| `skills/01_authentication_skill.md` | 3.2 Implement Authentication | JWT + refresh token, bcrypt, httpOnly cookies, token rotation |
| `skills/04_payments_skill.md` | 3.3 Implement Payments | Stripe checkout, webhooks, subscription lifecycle, access control |
| `skills/33_execution_testing.md` | 3.4 Write Tests That Catch Bugs | Test pyramid, given/when/then, unit/integration/E2E, factories, anti-patterns |
| `skills/34_execution_code_review.md` | 3.5 Review Code Like a Senior Engineer | 6 review dimensions, IDOR patterns, N+1 detection, feedback categorization |
| `skills/35_execution_debugging.md` | 3.6 Debug Any Problem Systematically | Reproduce → isolate → hypothesis → test → root cause → regression test |

---

## Skills: Category 4 -- Quality

| File | Skill | One-line summary |
|------|-------|-----------------|
| `skills/40_quality_security_review.md` | 4.1 Security Review | OWASP Top 10 as actionable code checks, IDOR, dependency audit, 25-item checklist |
| `skills/41_quality_performance.md` | 4.2 Performance Optimization | Measure first, bottleneck categories, EXPLAIN ANALYZE, N+1, caching, bundle analysis |
| `skills/42_quality_error_handling.md` | 4.3 Error Handling | Operational vs programmer errors, error hierarchy, global handler, external service failures |

---

## Skills: Category 5 -- Delivery

| File | Skill | One-line summary |
|------|-------|-----------------|
| `skills/50_delivery_git.md` | 5.1 Git and Version Control | Branch naming, Conventional Commits, atomic commits, PR template, undo safely |
| `skills/51_delivery_deployment.md` | 5.2 Deploy Software Safely | Pre-deploy checklist, migration safety, rolling/blue-green/canary, rollback triggers |
| `skills/52_delivery_monitoring.md` | 5.3 Monitor and Observe | Three pillars (logs/metrics/traces), four golden signals, alert design, incident process |
| `skills/53_delivery_documentation.md` | 5.4 Write Documentation That Gets Read | README structure, comment philosophy, API doc format, ADR, CHANGELOG, definition of done |

---

## Skills: Category 6 -- AI Agent Skills

| File | Skill | One-line summary |
|------|-------|-----------------|
| `skills/60_ai_vibe_coding.md` | 6.0 AI Vibe Coding | How to work with AI agents productively without drift, hallucination, or scope creep |
| `skills/61_ai_agent_instructing.md` | 6.1 AI Agent Instructing | How to write precise AI instructions and operating rules to prevent hallucination |

---

## Workflows

Step-by-step runbooks an AI agent or developer can follow end-to-end.

| File | Workflow | Purpose |
|------|----------|---------|
| `workflows/WF01_new_feature_workflow.md` | New Feature | Requirements → types → schema → service → API → tests → PR |
| `workflows/WF02_bug_fix_workflow.md` | Bug Fix | Reproduce → locate → hypothesis → failing test → fix → verify |
| `workflows/WF03_auth_implementation_workflow.md` | Auth Implementation | Schema → migration → service → routes → middleware → tests → security checklist |
| `workflows/WF04_payment_integration_workflow.md` | Payment Integration | Stripe client → schema → service → webhooks → API → access control → testing |
| `workflows/WF05_database_migration_workflow.md` | Database Migration | Classify → plan expand/contract → write migration → test → deploy safely → verify |
| `workflows/WF06_performance_optimization_workflow.md` | Performance Optimization | Define problem → instrument → identify bottleneck → fix → re-measure → deploy |
| `workflows/WF07_security_audit_workflow.md` | Security Audit | 8 categories: injection → auth → secrets → dependencies → headers → validation → errors → payments |
| `workflows/WF08_deployment_workflow.md` | Deployment | Pre-checks → announce → migrate → deploy → verify → smoke test → monitor |

---

## Situation Lookup Table

Find the right skill or workflow for any situation:

| Situation | Go to |
|-----------|-------|
| I just got a new task and need to understand it | `skills/10_mindset_read_task.md` |
| I need to break a large feature into tasks | `skills/11_mindset_break_problems.md` |
| I need to choose a tech stack | `skills/00_tech_stack_decision_matrix.md` |
| I need to make a technical decision and document it | `skills/12_mindset_technical_decisions.md` |
| I need to plan a new feature before building it | `skills/20_planning_feature.md` |
| I need to design a database schema | `skills/21_planning_database.md` |
| I need to design an API | `skills/22_planning_api.md` + `skills/03_api_design_skill.md` |
| I need to plan authentication | `skills/23_planning_auth.md` |
| I am implementing a full feature from scratch | `workflows/WF01_new_feature_workflow.md` |
| I need to implement auth from scratch | `workflows/WF03_auth_implementation_workflow.md` |
| I need to set up Stripe payments | `workflows/WF04_payment_integration_workflow.md` |
| I am investigating a bug | `workflows/WF02_bug_fix_workflow.md` |
| I need to write tests | `skills/33_execution_testing.md` |
| I am reviewing someone's code | `skills/34_execution_code_review.md` |
| I am debugging a hard problem | `skills/35_execution_debugging.md` |
| I need to do a security review | `skills/40_quality_security_review.md` + `workflows/WF07_security_audit_workflow.md` |
| Something is slow and I need to fix it | `skills/41_quality_performance.md` + `workflows/WF06_performance_optimization_workflow.md` |
| I need to improve error handling | `skills/42_quality_error_handling.md` |
| I am about to make a schema change | `skills/21_planning_database.md` + `workflows/WF05_database_migration_workflow.md` |
| I need to set up git branching and commits | `skills/50_delivery_git.md` |
| I am deploying to production | `skills/51_delivery_deployment.md` + `workflows/WF08_deployment_workflow.md` |
| I need to set up monitoring | `skills/52_delivery_monitoring.md` |
| I need to write documentation | `skills/53_delivery_documentation.md` |
| I am working with an AI coding agent | `skills/60_ai_vibe_coding.md` |
| I need to write AI agent instructions | `skills/61_ai_agent_instructing.md` |
| I need naming and structural conventions | `skills/00_project_conventions.md` |

---

## AI Agent Quick-Start Guide

If you are an AI coding agent using this library, read these four files first:

1. `skills/00_project_conventions.md` -- naming, structure, code conventions
2. `skills/61_ai_agent_instructing.md` -- your operating rules (read this and follow it)
3. The skill for the current category of task
4. The workflow for the specific task type (if one exists)

Operating rules summary (from `skills/61_ai_agent_instructing.md`):

- READ before acting: read every file you will modify before generating output
- VERIFY every import path, function reference, and database column against provided context
- NEVER invent library APIs, file paths, or utility functions
- STATE your scope before every response
- SIGNAL uncertainty explicitly with UNCERTAIN: or BLOCKED: -- do not guess
- DO only what was asked -- no unsolicited refactors, no scope creep
- MARK all placeholders: `// PLACEHOLDER: needs real value`
- CONFIRM what was done and what needs human verification at the end of every response

---

## File count summary

| Category | Count |
|----------|-------|
| Foundation | 2 |
| Mindset skills | 3 |
| Planning skills | 4 |
| Execution skills | 6 |
| Quality skills | 3 |
| Delivery skills | 4 |
| AI Agent skills | 2 |
| Workflows | 8 |
| **Total** | **32** |
