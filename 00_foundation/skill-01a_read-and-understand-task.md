# SKILL 01-A — How to Read and Understand Any Task Before Starting
Category: Foundations and Mindset
Applies to: All stacks, all task types, all AI coding agents

## What this skill covers
Reading a task is not the same as understanding it. The single most expensive mistake in software development is beginning implementation before the requirement is fully understood. This skill covers the exact process of extracting every constraint, assumption, success criterion, and scope boundary from any task description — whether it is a one-line Slack message or a 20-page PRD — so that work begins with complete clarity rather than optimistic interpretation.

## When to activate this skill
- At the very start of every single task, before writing or modifying any code
- When a requirement seems simple but contains implicit complexity
- When you receive a task with ambiguous scope (e.g. "add authentication")
- When working in an unfamiliar codebase or domain
- When the expected output format or acceptance criteria are not stated

## Core principles
1. **Understand before acting.** Starting without understanding compounds every mistake — you build the wrong thing faster.
2. **The task description is the contract.** Every word matters; missing a constraint means violating the contract.
3. **Explicit beats inferred.** Never assume what was meant — derive it from what was written, or ask one specific question.
4. **Define done before starting.** If you cannot state how you will verify success, you are not ready to start.
5. **Scope boundaries matter as much as scope.** What is NOT required is as important as what is.

## Step-by-step guide
1. Read the entire task description from start to finish without taking any action.
2. Identify the OUTPUT: what is the concrete deliverable? A function? A route? A UI component? A migration?
3. Identify the INPUT: what data, user actions, or system events trigger this feature?
4. List every explicit acceptance criterion — anything stated as "must", "should", "will", or "when X then Y".
5. List every implicit constraint — things the task assumes but did not state (e.g. "must work with existing auth system").
6. List every explicit exclusion — "do not…", "out of scope", "not part of this ticket".
7. Identify the failure states: what happens when input is wrong, the resource does not exist, or permissions are denied?
8. Identify dependencies: does this task require another task to be complete first? Does it change something other tasks rely on?
9. Write the definition of done in one sentence: "This task is complete when [X] given [Y] and [Z] is not broken."
10. If any of steps 2–8 has an unknown, write it as a specific question. Ask exactly one clarifying question — the most important one.

## The right way vs the wrong way

| Wrong | Right |
|-------|-------|
| Start coding as soon as you read the task | Read completely, extract constraints, then start |
| Assume "add auth" means JWT | Ask or infer from codebase what auth pattern is already in use |
| Treat acceptance criteria as optional guidance | Treat every acceptance criterion as a hard requirement |
| Begin without knowing what "done" looks like | Write done-definition before the first line of code |
| Discover edge cases halfway through implementation | Identify all failure paths during task reading |
| Ask multiple questions at once | Identify the single most-blocking unknown, ask only that |

## Stack-specific notes
**Node.js / TypeScript:** When reading tasks involving TypeScript, also identify the type contract — what types must be created or extended.
**Python:** Identify whether the task implies synchronous or async execution, as this affects the entire approach.
**Go:** Identify error handling expectations early — Go forces explicit error handling and the task may imply specific error types.
**All stacks:** The task reading step is identical across stacks. The output (done-definition) must always reference the specific language/framework's natural output format.

## Common mistakes
1. **Skimming instead of reading.** Fast reading misses the constraint buried in sentence three. Always read completely.
2. **Conflating "what" with "how".** The task states what to build; you decide how. Never let a task description dictate implementation unless it explicitly must.
3. **Assuming the happy path is the whole task.** Every task has at least one failure path. Find it during reading, not during testing.
4. **Ignoring implicit constraints.** "Add a user profile page" implicitly means: use the existing auth system, match the existing UI pattern, and not break any existing routes.
5. **Starting without a done-definition.** This leads to tasks that are "almost done" indefinitely.

## Checklist
- [ ] Task description read in full before any action taken
- [ ] Output deliverable identified and stated
- [ ] Input source and format identified
- [ ] All explicit acceptance criteria listed
- [ ] Implicit constraints identified
- [ ] Explicit exclusions noted
- [ ] Failure paths and error cases listed
- [ ] Dependencies identified
- [ ] Done-definition written in one sentence
- [ ] All unknowns listed; no more than one clarifying question sent
