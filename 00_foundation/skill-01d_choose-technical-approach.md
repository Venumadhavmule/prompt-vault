# SKILL 01-D — How to Choose Between Two Technical Approaches
Category: Foundations and Mindset
Applies to: All stacks, all architectural decisions, all AI coding agents

## What this skill covers
Every meaningful engineering decision has at least two viable paths. The skill is not in choosing the "best" option in isolation — there is rarely one — it is in choosing the option that best fits the specific constraints of this system, this team, and this moment. This skill covers a structured method for comparing any two technical approaches and arriving at a decision that is defensible, documented, and reversible where possible.

## When to activate this skill
- When choosing between two data modeling approaches
- When deciding between a library or building a custom solution
- When picking between REST, GraphQL, or tRPC for a new API
- When choosing a caching strategy
- Before any architectural decision that will be costly to reverse
- When a stakeholder is pushing for an approach you have doubts about

## Core principles
1. **Decide against explicit criteria, not intuition.** List what matters (performance, DX, complexity, reversibility) and evaluate both options against the same list.
2. **Bias toward reversibility.** All else equal, choose the option that is easier to change or remove later.
3. **Simple over clever.** A simpler solution that does the job beats an elegant one that adds complexity.
4. **Match existing patterns.** If the codebase already uses X, adding Y for the same purpose creates a split that costs more long-term than any short-term benefit of Y.
5. **Document the decision.** Any decision complex enough to require comparison deserves an ADR so the reasoning is not lost.

## Step-by-step guide
1. Write the decision statement: "We are deciding between [A] and [B] for [specific purpose]."
2. List the evaluation criteria: performance, developer experience, operational complexity, cost, reversibility, team familiarity, ecosystem maturity.
3. Build a comparison table: rows = criteria, columns = Option A / Option B. Score each cell (High/Medium/Low or 1-3).
4. Identify any hard blockers: a criterion where one option is disqualifying (e.g. "does not support our database" is a hard NO regardless of other scores).
5. Apply the "two-year test": in two years, if this turns out to be the wrong choice, how bad is the damage? Choose the option with the lower damage if wrong.
6. Check for prior art: has the team already made this decision for a similar system? If yes, match it unless you have a compelling technical reason not to.
7. State the chosen option with a one-paragraph rationale.
8. State explicit conditions under which you would revisit this decision.
9. Write an ADR if the decision affects the architecture or will be referenced by future engineers (see SKILL 01-F).

## The right way vs the wrong way

| Wrong | Right |
|-------|-------|
| Choose based on what you used last time | Evaluate against the specific requirements of this system |
| "Everyone uses X now so we should too" | Compare X to alternatives on the criteria that matter here |
| Make the decision without documenting it | Write an ADR for any non-trivial architectural decision |
| Choose the most powerful/complex solution | Choose the simplest solution that meets the requirements |
| Ignore existing patterns in the codebase | Match existing patterns unless there is a compelling reason not to |
| Decide then look for evidence to support it | List criteria first, then evaluate honestly against both options |

## Stack-specific notes
**Node.js:** Common decision points: Express vs Fastify vs Hono (choose Hono for edge/modern, Fastify for performance, Express for familiarity); Prisma vs Drizzle (Prisma for DX, Drizzle for raw SQL control); tRPC vs REST (tRPC for full-stack TS monorepos).
**Python:** FastAPI vs Django REST Framework (FastAPI for new greenfield async APIs; DRF for mature apps needing batteries-included admin/ORM); SQLAlchemy vs Tortoise ORM (SQLAlchemy for sync/complex queries, Tortoise for async).
**Go:** Standard library `net/http` vs gin vs chi vs fiber. For most cases, chi is the right answer unless you need the ecosystem of fiber.
**All stacks:** The decision framework (criteria table, two-year test) is language-agnostic.

## Common mistakes
1. **Deciding by familiarity alone.** "I know X" is not a criterion — it is bias. Factor it in as "team ramp-up time" but weigh it against the technical fit.
2. **Choosing the newer option because it is newer.** New means less battle-tested, fewer answers on Stack Overflow, and more breaking changes.
3. **Skipping the hard-blocker check.** An option that looks great on six criteria but has one dealbreaker should be eliminated early, not second-guessed later.
4. **Not documenting the decision.** Six months later, no one knows why X was chosen. This leads to the same decision being made again, sometimes differently.

## Checklist
- [ ] Decision statement written clearly (comparing A vs B for specific purpose)
- [ ] Evaluation criteria listed before comparing options
- [ ] Comparison table completed for both options
- [ ] Hard blockers identified and checked
- [ ] Two-year reversal test applied
- [ ] Existing codebase patterns checked for prior art
- [ ] Chosen option stated with a one-paragraph rationale
- [ ] Conditions for revisiting stated
- [ ] ADR written if decision is architectural (see SKILL 01-F)
- [ ] Decision did not default to "most familiar" without conscious justification
