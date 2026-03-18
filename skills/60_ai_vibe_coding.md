# SKILL 6.0 -- AI Vibe Coding: How to Work Effectively with AI Code Agents

## What this skill is

"Vibe coding" is iterative, AI-assisted development where you and an AI agent
build together in rapid cycles. It is extremely productive and extremely dangerous
at the same time. Sessions drift. Requirements erode. Generated code compiles but
does not match what you asked for. AI agents confidently write code that is 90%
right and 10% completely wrong, and they will not tell you which 10%.

This skill gives you a disciplined approach to AI-assisted coding that keeps the
session on track, catches failures before they accumulate, and ensures the code
produced actually solves the problem you started with.

---

## When to use this skill

- When asking an AI agent to implement a feature
- When using AI-generated code as a starting point and then iterating
- When a session has been running long and things feel "off"
- When AI output looks right but something seems wrong
- When you need to hand AI-generated code to a real team for review
- Whenever you are building something with stakes: production systems, user data, money

---

## Full Guide

### Core Principle: The AI Does Not Know Your System

Every AI model has a training cutoff. It does not know:
- Your database schema (unless you gave it)
- Your auth flow (unless you showed it)
- Your naming conventions (unless you shared them)
- Your error handling patterns (unless you provided examples)
- What your PR from last week changed

The moment you assume the AI "knows" your system is the moment it writes code
that compiles but silently does the wrong thing.

**Rule:** Before sending any task to an AI agent, explicitly give it the context it
needs. Never assume it remembers from earlier in the conversation.

---

### Step 1: Frame the task before you start

Before asking an AI to write any code, write down -- for yourself -- three things:

```
1. WHAT: What is the exact thing to be built?
2. ACCEPTANCE: How will I verify it is done correctly?
3. SCOPE: What is explicitly OUT of scope?
```

If you cannot write these three things clearly, you are not ready to prompt the AI.
Prompting an AI with a vague task produces vague code.

**Bad prompt:**
```
Add auth to my app
```

**Good prompt:**
```
Implement JWT-based login for existing users.

Context:
- User model is in prisma/schema.prisma (provided)
- Passwords are already hashed with bcrypt in the users table
- Access token: 15 min expiry, stored in memory on client
- Refresh token: 7 day expiry, httpOnly cookie
- Endpoint: POST /api/auth/login
- On success: return access_token, expires_in, and user object (id, email, firstName)
- On failure: return 401 with {"error": "INVALID_CREDENTIALS"}

Do NOT modify the User model. Do NOT add email verification. Do NOT add OAuth.

Success verification: I will call this endpoint with curl and verify the response shape.
```

---

### Step 2: Provide anchoring context explicitly

For every AI session that involves existing code, provide the relevant anchoring context:

**Context to always provide:**
- The schema or data model for any table being touched
- The existing function or file being modified (paste the current code)
- Your naming conventions if they are non-standard
- The specific error format your API uses
- Any shared utility the agent should use (e.g., `import { db } from "@/lib/db"`)

**How to provide context in long sessions:**
At the start of each new sub-task in a long session, restate:
```
"We are still working on X. The current state of [file] is [paste].
Now please do Y, using the same patterns."
```

This is not redundant. It is how you prevent context drift.

---

### Step 3: Verify incrementally -- not at the end

The most common failure mode in AI vibe coding: accept AI output all the way
through a feature, then run the code and find that half of it is wrong. 

Fixing accumulated drift is 5x more expensive than catching it early.

**Verification checkpoints:**

| After the AI generates... | Verify... |
|---|---|
| Database/schema changes | The schema is correct for the use case BEFORE running migration |
| A new function | Call the function mentally or with a test -- does it produce correct output? |
| An API endpoint | Match the request/response shape against the requirements |
| Any import | The import path actually exists in your codebase |
| A type | The type is used consistently -- no `as any` hacks |
| A test | The test actually tests a failure case, not just the happy path |

Stop every 3-4 generated files or after each logical unit (schema, service, controller, test).
Do a mini-review before continuing.

---

### Step 4: Detect drift before it compounds

AI sessions drift when:
- The AI loses context of earlier constraints
- You added clarifying messages that changed direction mid-session
- The AI started using a different pattern than the one it used in earlier files
- The task expanded in scope without explicit acknowledgment

**Drift detection signals:**
- Generated code uses `any` types where earlier code was strictly typed
- Function signatures in the service layer do not match how the controller called them
- Error response format changed between endpoints in the same session
- A new import appears that you did not provide (the AI invented a utility)
- The AI starts describing what it will do instead of doing it
- Variable naming style inconsistency (camelCase in some places, snake_case in others)

**When you detect drift:**
Do not continue. Stop the session and reframe:
```
"Stop. Before continuing, here is what we have established so far: [summary].
Here are the constraints we agreed on: [list].
Now continue with [next task] while keeping all of these in place."
```

---

### Step 5: Treat AI output as an unreviewed PR

Every file the AI generates is an unreviewed pull request.
Apply the same standards you would to a junior developer's first PR.

Review checklist for every AI-generated file:

```
[ ] Does it do what the task actually asked?
[ ] Does it use the actual import paths from my project?
[ ] Are there any `// TODO` or placeholder comments left in?
[ ] Does it handle error cases, not just the happy path?
[ ] Is there any hardcoded value (URL, secret, ID) that should be an env var?
[ ] Does the function name match what it does?
[ ] Is there any `console.log` or debug code left in?
[ ] Are types correct, or is anything smuggled in with `as any`?
[ ] Would this code pass the existing linter/type-checker?
```

---

### Step 6: Manage scope actively

Vibe coding sessions expand naturally. The AI suggests a helper. You say "yes."
The AI adds an extra endpoint. You say "sure." Before long you are building three
features instead of one, and the session is 12,000 tokens deep and untracked.

**Rules:**
1. Every new thing the AI proposes that was not in the original plan is a scope addition
2. Write the scope addition down. Decide consciously: YES (add to this session) or BACKLOG
3. If you add scope, acknowledge it: "We are now adding X. Our scope is now: [list]."
4. If a session has been running more than 45 minutes, summarize and start fresh

**Session restart prompt:**
```
We have been working on [feature]. Here is what has been completed so far:
[list of completed files]

Here is what remains:
[list of remaining tasks]

Let's continue with [next task].

Key constraints that must be maintained:
- [constraint 1]
- [constraint 2]
```

---

### Step 7: Terminal verification (the real test)

AI output that looks right in the chat is not confirmed working until it runs.

For every session, before calling it done:
1. Run the type-checker: `tsc --noEmit`
2. Run the linter: `eslint .`
3. Run the tests: `pnpm test`
4. Run the actual scenario manually (curl the endpoint, click through the UI)

If any of these fail, do not ask the AI to fix it blindly.
Paste the exact error to the AI with the relevant file and say:
```
"Running [command] gives this error: [exact error output].
Here is the file that needs to change: [file content].
Fix only the error. Do not change anything else."
```

---

## What to avoid

DO NOT paste AI-generated code without reading it. 100% copy-paste is not a
development process, it is slot-machine engineering.

DO NOT let a session run for more than 60 minutes without a summary checkpoint.
Long sessions degrade context quality and accumulate undetected drift.

DO NOT accept "I'll add that later" from an AI agent. If the task needs error handling,
that is not optional. Ask for it now or explicitly mark the omission as a known gap.

DO NOT trust that an AI-invented import path exists. Run `ls` or check the actual
file structure before trust.

DO NOT use `as any` to make AI-generated types compile. That is not fixing, that is hiding.

DO NOT assume the AI's most recent output is consistent with its earlier outputs.
Cross-check the shapes and patterns across generated files.

---

## Checklist

- [ ] Task is fully defined (WHAT, ACCEPTANCE, SCOPE) before prompting
- [ ] All required context provided (schema, existing code, conventions)
- [ ] Scope boundaries stated explicitly in the prompt
- [ ] Verified schema/types before running any migration
- [ ] Reviewed each generated file as if it is an unreviewed PR
- [ ] Import paths verified against actual codebase structure (no invented paths)
- [ ] No hardcoded secrets, URLs, or IDs
- [ ] No leftover `// TODO` or placeholder code
- [ ] No `as any` type suppressions
- [ ] No `console.log` debug statements in production code
- [ ] Drift check done after every 3-4 files or logical unit
- [ ] Type checker passes: `tsc --noEmit`
- [ ] Linter passes: `eslint .`
- [ ] Tests run and pass
- [ ] Manual scenario tested end-to-end before marking done
