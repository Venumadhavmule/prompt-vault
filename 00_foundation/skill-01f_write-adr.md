# SKILL 01-F — How to Write an Architecture Decision Record (ADR)
Category: Foundations and Mindset
Applies to: All stacks, all teams, any significant technical decision

## What this skill covers
Architecture Decision Records are how teams preserve the reasoning behind decisions so future engineers do not have to reverse-engineer thinking from code alone. Without ADRs, decisions get revisited repeatedly, wrong assumptions compound silently, and departures from past decisions happen without awareness of the original reasoning. This skill covers exactly how to write an ADR that is useful, not bureaucratic — one that a new engineer can read in three minutes and understand not just what was decided, but why, what was rejected, and what would make this decision change.

## When to activate this skill
- When choosing a database, ORM, or data layer for a new service
- When making a decision that affects the overall architecture (e.g. microservices vs monolith)
- When adopting a new framework, library, or third-party integration
- When choosing an auth strategy
- When deciding on an API contract approach (REST vs GraphQL vs tRPC)
- Any time the decision could reasonably be made differently by two engineers looking at the same requirements

## Core principles
1. **Status matters.** An ADR is always in one of these states: Proposed, Accepted, Deprecated, Superseded. Keep status current.
2. **Alternatives rejected are as important as the chosen option.** If you do not document why alternatives were rejected, the next person will re-evaluate them and waste time.
3. **Context first, then decision.** An ADR without context is just an assertion. The context section is the most important part.
4. **ADRs are immutable when accepted.** Do not edit an accepted ADR to change the decision — create a new ADR that supersedes it.
5. **Short and usable beats long and complete.** If no one reads the ADR, it has failed. Aim for one document a person can read in 3 minutes.

## Step-by-step guide
1. Create a new file in `docs/architecture/` (or your configured ADR directory) with the naming convention: `ADR-NNNN-short-title.md` where NNNN is a zero-padded sequential number.
2. Set the status to **Proposed** initially.
3. Write the **Context** section: what is the situation, constraint, or problem that requires a decision? What forces are at play (performance, cost, timeline, existing systems)?
4. Write the **Decision** section: what is the chosen approach, stated as a fact ("We will use..."), not a recommendation ("We should consider...").
5. Write the **Rationale** section: why this? What criteria were used? What makes this option better than alternatives for this specific context?
6. Write the **Alternatives Considered** section: list each major alternative with 2-3 sentences explaining why it was not chosen.
7. Write the **Consequences** section: what changes now? What becomes easier? What becomes harder? What technical debt, if any, does this incur?
8. Add a **Review date or trigger** — circumstances under which this decision should be revisited (e.g. "revisit if user count exceeds 1M", "revisit if Prisma adds native streaming support").
9. Have the ADR reviewed by at least one other engineer before changing status to **Accepted**.
10. Link the ADR from the relevant code with a comment: `// See ADR-0023 for why this pattern is used`.

## The right way vs the wrong way

| Wrong | Right |
|-------|-------|
| Edit an accepted ADR when the decision changes | Create a new ADR with status "Supersedes ADR-N" |
| Write the decision without the alternatives | Document every major alternative and why it was rejected |
| "We chose PostgreSQL." (no context) | "Given our relational data model, need for ACID transactions, and existing team PostgreSQL expertise, we chose PostgreSQL over MongoDB." |
| Write the ADR after the code is already shipped | Write the ADR at the point of decision, before or alongside the implementation |
| Store ADRs in a Notion page no one links to | Store in the repository alongside the code it documents |
| Leave ADR status as "Proposed" forever | Update status: Proposed → Accepted → Deprecated/Superseded |

## Stack-specific notes
**Node.js / TypeScript:** Common ADR topics: ORM selection, API framework, monorepo tooling, state management approach, testing framework. Store in `docs/architecture/`.
**Python:** Common ADR topics: async vs sync runtime, ORM choice, Pydantic version, deployment target. The `adr-tools` CLI works well for automated ADR numbering.
**Go:** Less common to need ADRs for language-level choices (Go is opinionated), but useful for: service architecture, gRPC vs REST, chosen router, config management approach.
**All stacks:** The ADR format is language-agnostic. Store them in version control alongside the code they document.

## Common mistakes
1. **Writing ADRs as justifications rather than records.** An ADR is not a defense brief — it is an honest account of the decision, including its downsides.
2. **Not dating the ADR.** Context changes over time. A date tells readers whether the constraints that drove the decision still apply.
3. **Making the context too vague.** "We needed something scalable" is not context. "We expected 500K daily active users within 18 months, requiring the database to handle 10K concurrent writes" is context.
4. **Skipping the consequences section.** The consequences section is where the ADR's author is honest about trade-offs. It prevents future engineers from being blindsided.

## Checklist
- [ ] File named with sequential number: `ADR-NNNN-short-title.md`
- [ ] Stored in `docs/architecture/` or equivalent in-repo location
- [ ] Status set: Proposed / Accepted / Deprecated / Superseded
- [ ] Date included
- [ ] Context section written with specific constraints, not vague generalities
- [ ] Decision stated as a fact, not a recommendation
- [ ] Rationale written with specific criteria
- [ ] At least two alternatives documented with reasons for rejection
- [ ] Consequences section honest about trade-offs and new constraints introduced
- [ ] Review trigger condition stated
- [ ] Reviewed by at least one other engineer before status set to Accepted
- [ ] Relevant code references this ADR with a comment
