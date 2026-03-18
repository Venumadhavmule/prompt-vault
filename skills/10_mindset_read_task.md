# SKILL 1.1 -- How to Read and Understand Any Task Before Starting

## What this skill is

This skill covers the single most important habit that separates senior engineers
from junior ones: the discipline of fully understanding a task before touching
any code or tool. Misunderstood tasks produce correct implementations of the wrong
thing. This skill ensures you build exactly what was asked, catch ambiguities early,
and never waste time executing in the wrong direction.

Both human developers and AI coding agents must apply this skill at the start of
every task, no exceptions. The three minutes spent here save three hours of rework.

---

## When to use this skill

- At the very start of any task before writing a single line of code
- When a task description is long, complex, or uses vague language
- When you feel uncertain about what "done" looks like
- When a task references systems, codebases, or constraints you have not fully read
- When you are about to make an assumption that changes the scope significantly
- Whenever an AI agent is given a new instruction or user request

---

## Full Guide

### Step 1: Read the entire task description once without acting

Read everything first. Do not start typing. Do not start planning. Do not open any
file. Just read to the end. This is the single most violated rule in software development.
Developers and agents start executing after reading the first sentence.

Force yourself to read the complete description before forming any opinion about
how to solve it.

---

### Step 2: Extract the three core elements

After reading, identify and write down:

**1. The Goal**
What is the desired outcome in one sentence?
Ask: "What business or user problem does this solve?"
Example: "Users must be able to reset their password via email."

**2. The Constraints**
What limits apply? This includes:
- Technology constraints (must use existing stack, cannot change DB schema, etc.)
- Time constraints (must be done before a migration, before a release)
- Behavioral constraints (must not break existing functionality, must be backward compatible)
- Security constraints (must follow existing auth pattern, must validate input the same way)
- Performance constraints (must respond in under 200ms, must not add N+1 queries)

**3. The Definition of Done**
What specific, observable conditions confirm this task is complete?
Example:
- A user can request a reset from /forgot-password
- They receive an email with a link that expires in 1 hour
- Clicking the link shows a form to set a new password
- Submitting a valid password updates their credentials and invalidates the token
- The old password no longer works after reset

If you cannot write a Definition of Done, you do not understand the task yet.

---

### Step 3: Classify the task type

Knowing the task TYPE changes how you approach it. Classify it as one of:

| Type | What it is | How to approach it |
|---|---|---|
| New Feature | Adds new functionality that did not exist | Plan data model first, then API, then UI |
| Bug Fix | Corrects unintended behavior | Reproduce first, root cause second, fix third |
| Refactor | Changes structure without changing behavior | Define what must stay the same (tests prove it) |
| Design / Planning | Produces a plan, not code | Focus on trade-offs and Documentation |
| Infrastructure | Changes deployment, CI, or config | Test on non-prod first. Document rollback. |
| Code Review | Reviews someone else's code | Apply the 6-dimension review framework |
| Performance | Improves speed without changing behavior | Measure first. Never optimize by instinct. |
| Security Fix | Addresses a vulnerability | Check for similar patterns across entire codebase |

State the type explicitly before continuing.

---

### Step 4: Identify what you do NOT know yet

Make a list of unknowns. For each unknown, decide:
- Is this a BLOCKING unknown? (cannot proceed without answering it)
- Is this an ASSUMPTION I can state and proceed with?
- Is this a DETAIL I can discover by reading the codebase?

Example unknowns:
- "Does the user table already have a password_reset_token column?" -> DISCOVER: read schema
- "How long should the reset token be valid?" -> ASSUMPTION: 1 hour unless specified
- "Should the email be queued or sent synchronously?" -> BLOCKING: ask if not obvious

For blocking unknowns, proceed to Step 5.
For assumptions, write them down explicitly before proceeding.
For discoverable details, go read the relevant code or schema NOW before continuing.

---

### Step 5: The single clarifying question rule

If you have blocking unknowns, you are allowed exactly ONE clarifying question.

**How to pick the right question:**
Ask the most blocking question -- the one whose answer determines the most other
decisions. Do not ask multiple questions at once. If you have 5 unknowns, figure out
which one, once answered, resolves or allows you to assume the others.

**Good question format:**
"Before I start, one thing I need to confirm: [specific question with context about why it matters]?"

**Bad question format (never do this):**
"I have several questions: 1) Should I use JWT or sessions? 2) Do you want me to update the DB schema?
3) What email should the reset come from? 4) Should I add rate limiting?"

The single question rule forces you to think deeply about what actually blocks you.
Most of the time, you can figure it out yourself.

---

### Step 6: Restate the task in your own words

Before starting, write a one-paragraph summary of what you are about to do.
This is not for the user -- it is for you. Writing it forces you to confront any
gaps in your understanding.

Template:
```
I understand the task as follows: [goal in one sentence]. To accomplish this,
I will need to [list the key steps you expect to take]. I am assuming [list assumptions].
Done means [your definition of done]. I will know I have succeeded when [observable outcome].
```

If you cannot fill out this template clearly, you are not ready to start.

---

### Step 7: The "reverse engineer from done" technique

Start from what finished looks like and work backwards.

1. Describe the final state as specifically as possible
   (What UI does the user see? What DB records exist? What API response is returned?)
2. Ask: "What must exist in the layer just below the surface to produce this?"
3. Repeat until you reach the current state of the codebase

Example: Password reset done state = user sees "Password changed successfully" and is logged in
- What produced that? The frontend confirmed the API returned 200 with a new auth token
- What produced that? The API validated the token, updated the password hash, returned a JWT
- What produced that? A reset_tokens table entry was valid, not expired, and matched the hash
- What produced that? An email was sent with a link containing the token
- What produced that? The user submitted their email to /forgot-password

This technique reveals every step you must implement in the correct order.

---

### Step 8: Signs you have misunderstood a task

Stop and re-read the task if you notice any of these:

- You cannot write a Definition of Done in concrete, testable terms
- Your implementation requires more than 3x the effort you estimated
- You are making a change to a part of the system that was not mentioned in the task
- You realize your solution would break existing functionality that was not in scope
- You are halfway through and realize you need clarification that should have been gathered upfront
- You implemented something that "makes sense" but you cannot trace it back to a specific requirement
- The person who assigned the task would not recognize your deliverable as what they asked for

When any of these appear: stop, re-read, restate, clarify before continuing.

---

### Step 9: The pre-task checklist

Fill this out before starting every task:

```
TASK: [paste the task title here]

Goal (one sentence):
__________________________________________________

Task type (feature/bug/refactor/design/infra/review/perf/security):
__________________________________________________

Definition of Done (bullet list of observable conditions):
-
-
-

Constraints (technology, time, behavior, security, performance):
-
-
-

What I need to discover in the codebase before starting:
-
-

Assumptions I am making (state them explicitly):
-
-

Blocking questions (one only, if needed):
__________________________________________________

My restatement (the reverse-engineer-from-done summary):
__________________________________________________
```

---

## What to avoid

DO NOT start writing code immediately after reading the first sentence.

DO NOT treat tasks as so obvious they need no analysis -- this causes the most bugs.

DO NOT list multiple clarifying questions. Pick the one that matters most.

DO NOT make assumptions silently. Every assumption you make must be written down.

DO NOT use vague definitions of done like "the feature works." Write observable, testable conditions.

DO NOT proceed if you cannot state what done looks like. It means you do not understand the task.

DO NOT redefine the task to fit what you already know how to do. Match the requirement.

DO NOT skip the restatement step. It is the most important validation of your understanding.

---

## Checklist

Before starting any task, verify:

- [ ] I have read the entire task description without acting
- [ ] I have written the goal in one sentence
- [ ] I have classified the task type
- [ ] I have written a concrete, testable Definition of Done
- [ ] I have listed all constraints
- [ ] I have identified what to discover in the codebase (and gone to read it)
- [ ] I have listed all assumptions explicitly
- [ ] I have asked at most one clarifying question if truly blocked
- [ ] I have restated the task in my own words
- [ ] I have applied the reverse-engineer-from-done technique
- [ ] I have filled out the pre-task checklist
- [ ] I would recognize my finished deliverable as matching what was asked
