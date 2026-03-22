# SKILL 1.3 -- How to Make Technical Decisions with Confidence

## What this skill is

This skill teaches a repeatable, evidence-based process for making technical decisions
at any level: library choice, architecture pattern, data model design, or infrastructure
strategy. Engineering decisions made by instinct or convention without explicit
trade-off analysis become technical debt, missed requirements, or architectural
regrets within 12 months. This skill replaces instinct with a structured process
that produces decisions you can defend, document, and revisit.

---

## When to use this skill

- Choosing between two or more technical approaches, libraries, or tools
- Deciding on an architecture pattern (monolith vs microservices, REST vs tRPC, etc.)
- Making a build vs buy vs open-source call
- Evaluating whether to refactor or rebuild an existing system
- Responding to a stakeholder who wants a different technical direction
- Any decision whose consequences will persist for more than 3 months

---

## Full Guide

### Step 1: State the decision explicitly

Before anything else, write the decision in one sentence.

Template: "We need to decide: [decision statement]"

Examples:
- "We need to decide: should we use Prisma or Drizzle ORM for database access?"
- "We need to decide: should we implement an in-house auth system or use Auth.js?"
- "We need to decide: should we split the API into microservices now or stay with a monolith?"

If you cannot write the decision in one sentence, you have not yet clearly identified
what decision needs to be made. Do that first.

---

### Step 2: Identify the options

List every realistic option including the "do nothing" or "keep current" option.
Usually there are 2-4 options. If you have more than 5, group the similar ones.

Do not start evaluating yet. Just list.

---

### Step 3: The trade-off table method

For each decision, create a trade-off table:

| Criteria | Option A | Option B | Option C |
|---|---|---|---|
| [Criterion 1] | [Assessment] | [Assessment] | [Assessment] |
| [Criterion 2] | [Assessment] | [Assessment] | [Assessment] |

**Standard evaluation criteria for library/tool decisions:**

| Criteria | What to check |
|---|---|
| Maturity | How old is it? Is v1.0 stable? Has it been used in production at scale? |
| Maintenance | When was the last commit? Are issues being responded to? Is there a paid company behind it? |
| Community | How many weekly downloads? Stack Overflow questions answered? Discord/Slack active? |
| Developer Experience (DX) | How long does it take a new developer to become productive? Is the API intuitive? Is the docs good? |
| Performance | What are the benchmarks for your use case? Does it matter for your scale? |
| Security | Is the package audited? Are CVEs being patched? Is the surface area small? |
| License | MIT/Apache/BSD = safe. GPL = check if network use triggers copyleft. Custom = legal review required. |
| Bundle size | Relevant for frontend. Check bundlephobia.com |
| Type safety | Does it have first-class TypeScript support? Or just community-maintained types? |
| Test-ability | Can behavior be unit-tested without a running server/DB/service? |
| Ecosystem fit | Does it integrate cleanly with the rest of your stack? |
| Escape-hatch cost | How hard is it to replace this in 2 years if it becomes a problem? |

You do not need all criteria for every decision. Pick the 4-6 that are most relevant
to your context.

---

### Step 4: Identify the deciding factor

Every decision has one criterion that matters most for this specific context.
Name it explicitly: "The deciding factor for this decision is: [criterion] because [reason]."

Examples:
- "The deciding factor is escape-hatch cost because we are early-stage and may pivot."
- "The deciding factor is DX because our team is small and onboarding speed matters."
- "The deciding factor is security because this handles PII in a HIPAA context."

The deciding factor is the tiebreaker when options score similarly across other criteria.

---

### Step 5: Build vs Buy vs Open Source decision

This is the most common meta-decision. Apply this framework:

**Build in-house when:**
- The functionality is your core competitive advantage
- No existing solution fits your specific data model or workflow
- You have the team capacity to maintain it long-term
- The external solution is too expensive at your scale

**Buy (SaaS/commercial) when:**
- The functionality is infrastructure (auth, payments, email, search, video)
- Compliance requirements (HIPAA, PCI) are easier to meet via a certified vendor
- Speed to market matters more than cost
- The ongoing maintenance cost of building exceeds the SaaS price

**Use open source when:**
- A mature, maintained solution exists that fits your needs
- You can contribute back or fork if needed
- The license is compatible with your business model
- The security surface has been reviewed by the community

**The default rule:**
For payments: use Stripe. For auth: use a proven library (Auth.js, Clerk) unless you have a specific reason to build.
For email: use a transactional service (Resend, Postmark). For search: use a proven engine (Algolia, Typesense, PostgreSQL FTS).
Only build in-house when it is your differentiator.

---

### Step 6: Make the decision with incomplete information

You will rarely have perfect information. Here is how to proceed:

1. List what you do know
2. List what you do NOT know (call these "information gaps")
3. For each gap, assess: would the answer to this change the decision? If yes, go get the answer.
   If no, proceed without it.
4. For gaps you cannot fill, state the assumption you are making and what would need to be true
   for it to be wrong
5. Make the decision as reversible as possible: "If assumption X turns out to be wrong,
   the cost to reverse this decision is [estimate]"

A good decision under uncertainty is: the option that is most correct given current
information AND the easiest to reverse if new information changes the picture.

---

### Step 7: Document the decision as an ADR

Architecture Decision Records (ADRs) are the most important documentation a team can maintain.
They record not just WHAT was decided but WHY, which is what matters when revisiting 12 months later.

**ADR format:**

```markdown
# ADR-[number]: [Title]

**Date**: [YYYY-MM-DD]
**Status**: [Proposed | Accepted | Deprecated | Superseded by ADR-N]
**Deciders**: [Names or roles of who made this decision]

---

## Context

[1-3 paragraphs: What situation or problem forced this decision?
What constraints exist? What is at stake?]

## Decision

We will [chosen option].

## Options Considered

### Option A: [Name]
**Pros**: [bullet list]
**Cons**: [bullet list]

### Option B: [Name]
**Pros**: [bullet list]
**Cons**: [bullet list]

## Trade-off Table

| Criteria | Option A | Option B |
|---|---|---|
| [Criterion] | [Score/note] | [Score/note] |

## Deciding Factor

[One sentence: why Option X won over Option Y]

## Consequences

**Positive**: [what this enables]
**Negative / Trade-offs accepted**: [what we give up]
**Risks**: [what could go wrong and how we mitigate]

## Review Trigger

[Condition that should prompt revisiting this ADR, e.g., "if we exceed 100 req/s" or "when the team grows past 10 engineers"]
```

Store ADRs in `docs/architecture/ADR-001-[slug].md`.
Number them sequentially. Never delete an ADR, only change its status.

---

### Step 8: The "2-year regret" filter

Before finalizing any major architectural decision, ask:

"In 2 years, if this decision turns out to be wrong, how bad will the regret be?"

Specifically:
- How much code will need to change?
- How much data migration will be required?
- How much will it cost to reverse or migrate?
- How many developers will be affected during the reversal?

If the regret score is HIGH, only proceed if you have strong conviction AND the
decision is not reversible without major cost. Find a way to make it more reversible.

If the regret score is LOW, make the decision now and move on. Spend your analysis
budget on high-regret decisions.

---

### Step 9: How to push back on a bad technical decision

When a stakeholder or manager wants a technical approach you believe is wrong:

1. Do not push back with opinion. "I don't like that approach" loses every time.
2. Build a trade-off table first. Show it visually.
3. Agree on the evaluation criteria BEFORE presenting the conclusion.
   "Can we agree that performance and maintainability are the primary criteria here?" → Yes.
   Then show how the preferred option performs on those agreed criteria.
4. Present the cost of the proposed approach in concrete terms:
   "This approach means every time we change X, we also have to manually update Y and Z,
   which based on our current release cadence is about 3 hours of extra work per week."
5. Propose your alternative clearly: "I'd recommend [option] because it solves the same
   problem AND eliminates that maintenance cost. Here is the trade-off table."
6. If they still choose the approach you disagree with: voice your concern formally in writing
   (in the ADR under "Risks"), then execute the decision faithfully and without sabotage.

---

## What to avoid

DO NOT make decisions based on personal preference without stating the trade-offs.

DO NOT pick the newest or most hyped technology without checking maturity and maintenance.

DO NOT build in-house what a well-maintained, trusted library or service already provides.

DO NOT make reversibility-destroying architectural decisions under time pressure without documenting the risk.

DO NOT document decisions only as code comments. Use ADRs.

DO NOT make the same decision arguments twice. Write an ADR the first time so it never needs to be re-litigated.

DO NOT skip the "2-year regret" filter for architectural decisions.

DO NOT push back on technical decisions with opinion. Use data, trade-off tables, and concrete cost estimates.

---

## Checklist

Before finalizing any technical decision, verify:

- [ ] I have stated the decision in one sentence
- [ ] I have listed all realistic options including "do nothing"
- [ ] I have built a trade-off table using relevant criteria
- [ ] I have named the deciding factor explicitly
- [ ] I have applied the build vs buy vs open source framework if relevant
- [ ] I have listed my information gaps and decided whether they are blockers
- [ ] I have assessed how reversible this decision is
- [ ] I have applied the 2-year regret filter for major architectural decisions
- [ ] The decision is documented in an ADR
- [ ] The ADR includes consequences, both positive and negative
- [ ] The ADR includes a review trigger for when to revisit
- [ ] If pushing back on a stakeholder decision, I am using trade-offs and cost estimates, not opinion
- [ ] The ADR is stored in `docs/architecture/` and numbered sequentially
