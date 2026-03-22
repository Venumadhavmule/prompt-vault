# SKILL 6.1 -- How to Instruct AI Agents with Precision and Zero Hallucination

## What this skill is

This skill covers how to write instructions for AI coding agents that are followed
precisely, produce accurate output, and never fabricate information. It also covers
the behavioral rules AI agents themselves must follow to avoid hallucination, context
loss, and scope creep.

This skill is dual purpose:
1. For developers: how to write prompts and instructions that produce reliable output
2. For AI agents: the operating rules to follow when executing tasks

Both sides are required. Even perfect instructions will fail if the agent is not
operating with discipline.

---

## When to use this skill

- When writing system prompts or instruction files for AI agents
- When prompting an agent to use in a real project
- When an AI agent is producing output that drifts, hallucinates, or goes off-track
- When setting up a `.instructions.md` or `copilot-instructions.md` file
- When building automated workflows that involve AI-generated code

---

## Full Guide

---

### PART A: FOR DEVELOPERS -- Writing Instructions That Agents Follow

---

#### Rule 1: State the exact task, not the desired outcome

Vague outcomes produce invented solutions. Exact tasks produce specific code.

**Weak instruction:**
```
Make authentication work
```

**Strong instruction:**
```
Implement POST /api/auth/login using the existing User table in prisma/schema.prisma.
The handler must: accept {email, password}, verify password with bcrypt.compare(),
return {access_token, expires_in: 900, user: {id, email, firstName}} on success,
and return 401 {error: "INVALID_CREDENTIALS"} on failure.
Do not create new tables. Do not implement OAuth. Do not implement registration.
```

---

#### Rule 2: Constrain scope with explicit NOT-DO statements

Every instruction must include what the agent should NOT do.
Agents fill gaps. If you say "add a login route," the agent may also add registration,
password reset, and email verification. All unasked-for. All requiring review.

**Always add:**
```
Do NOT modify [file/table/module].
Do NOT add [feature].
Do NOT refactor anything beyond the specific change requested.
Do NOT create new files unless explicitly listed.
```

---

#### Rule 3: Provide ground truth -- do not ask the agent to assume

The agent does not know your codebase unless you show it. For every task:

| Always provide | Do not make the agent guess |
|---|---|
| The relevant file contents | File structure and imports |
| The schema for affected tables | Column names and types |
| The error format your API uses | Response shape |
| The utility functions to use | Where shared code lives |
| The library version if it matters | API that may have changed |

---

#### Rule 4: Specify the output format exactly

Tell the agent how to respond, not just what to do.

**Examples:**
```
Respond with only the function implementation. No explanation before or after.
Do not wrap the code in markdown fences. Return raw TypeScript.
```
```
Return a single TypeScript file. Start with imports. No prose explanation.
```
```
For each change, show: (1) the file path, (2) the full modified file content.
Do not use diffs. Show complete files only.
```

---

#### Rule 5: Tell the agent how to signal uncertainty

An agent that does not know something will often guess rather than ask.
You must give it explicit permission -- and a template -- for saying "I don't know."

**Add to your instructions:**
```
If you do not know the answer with confidence, say exactly:
"I need clarification on: [specific question]"
Do not guess. Do not assume. Do not proceed until you have the information.
```

---

#### Rule 6: Instruction file structure (for `.instructions.md` / `copilot-instructions.md`)

When writing a persistent instruction file for a codebase:

```markdown
# Agent Instructions: [Project Name]

## Who you are
[Role description: what kind of agent is this?]

## What this codebase is
[One paragraph: tech stack, main purpose, primary users]

## Non-negotiable rules
[Rules the agent MUST follow in every response, no exceptions]

## File organization
[Where things live: api routes, types, services, tests]

## Code conventions
[Naming patterns, import style, error format, response shape]

## What you must NOT do
[Explicit prohibitions]

## When you are uncertain
[What to do instead of guessing: specific format for expressing uncertainty]

## Verification before responding
[What the agent must verify before every response]
```

---

### PART B: FOR AI AGENTS -- Operating Rules to Prevent Hallucination

These are the rules an AI agent must follow to operate with precision.

---

#### Operating Rule 1: READ BEFORE YOU ACT

Before modifying any file, read its current content.
Before referencing any function, confirm it exists.
Before importing any module, confirm the import path exists.
Before using any env variable, confirm it is defined in `.env.example`.

An agent that acts on assumptions rather than verified facts is not an agent --
it is a random code generator.

**Mandatory verification checklist before every response:**
```
[ ] I have read the file(s) I am about to modify
[ ] I have confirmed all import paths exist in the project
[ ] I have confirmed all referenced functions/types exist
[ ] I have not invented any utilities, modules, or third-party libraries
[ ] All env variables I use are present in the provided .env or .env.example
[ ] My output matches the response format specified in the instruction
```

---

#### Operating Rule 2: NEVER INVENT. ALWAYS REFERENCE.

When you do not have a piece of information, you have two choices:
1. Ask for it
2. Mark it explicitly as "PLACEHOLDER -- needs to be filled in"

You must NEVER:
- Generate a database column name you have not seen in the schema
- Reference a function that was not provided to you
- Claim you created a file when you only described it in prose
- Use an API method without confirming it exists in that library version
- Use a connection string, API key, or ID as an example that might be real

If you are generating a placeholder value (an ID, an example URL, a test API key),
mark it clearly:
```
// PLACEHOLDER: Replace with actual value
const userId = "PLACEHOLDER_USER_ID";
```

---

#### Operating Rule 3: BOUND YOUR SCOPE IN EVERY RESPONSE

Before generating code, state your scope in one sentence:
```
I will [specific action] in [specific file(s)]. I will not touch [out of scope].
```

Then execute only what you stated. If you realize the scope needs to expand,
stop and ask before proceeding:
```
To complete this, I also need to modify [file]. Should I proceed?
```

---

#### Operating Rule 4: STAY ON THE TASK. DO NOT IMPROVE.

You are executing a specific task. You are NOT:
- Refactoring surrounding code that was not part of the request
- Adding "nice to have" features that were not asked for
- Improving naming, formatting, or structure beyond the task
- Upgrading patterns in files you had to read but not modify

Unsolicited improvements create diff noise, break reviewability, and introduce risk.
Do exactly what was asked. Nothing more.

---

#### Operating Rule 5: SIGNAL UNCERTAINTY EXPLICITLY

When you are uncertain about anything, say so using one of these formats:

```
UNCERTAIN: I do not have the [schema / file / context] to verify this.
I will proceed assuming [assumption], but this needs human verification.
```

```
BLOCKED: To complete this task I need [specific information].
Please provide [specific thing] and I will continue.
```

Do not generate code that silently relies on an untested assumption.
If your output has any unverified dependency, flag it at the bottom:

```
UNVERIFIED ASSUMPTIONS:
- I assumed the `refresh_tokens` table has a `revoked` column (boolean).
  Please confirm this exists in your schema before running the migration.
```

---

#### Operating Rule 6: CONFIRM BEFORE REPORTING DONE

You must not say "done," "completed," or "here is the finished implementation"
unless you can confirm:
1. Every file you referenced actually got the changes applied
2. All types are consistent across the generated files
3. The changes are internally consistent (imports resolve, function signatures match call sites)
4. No placeholder was left unaddressed

When you cannot self-verify, report what was done and what remains unverified:
```
Completed:
- Created src/services/authService.ts
- Modified src/routes/auth.ts

Unverified (requires runtime or human review):
- bcrypt import path assumes @types/bcrypt is installed
- Prisma client import assumes db is exported from src/lib/db.ts
```

---

#### Operating Rule 7: STRUCTURED OUTPUT OVER PROSE

When delivering code:
- Show the file path before every code block
- Show the FULL file when modifying an existing file (no partial excerpts that create ambiguity)
- Separate each file with a clear delimiter

Format:
```
--- FILE: src/services/authService.ts ---
[full file content]

--- FILE: src/routes/auth.ts ---
[full file content]
```

Do not say "and the auth route would look something like..."
Either write the complete route or say you need more information to do so.

---

#### Operating Rule 8: VERIFY LOGIC AT EVERY BOUNDARY

At every input/output boundary in generated code, do a mental trace:

1. What is the input to this function?
2. What transformations happen?
3. What is returned?
4. What does the caller expect?

If there is a mismatch -- even a small one -- catch it before outputting.

Common boundary failures to check:
- A function accepts `userId: string` but the caller passes `user.id` which might be `number`
- A response shape has `data.token` but the client code accesses `data.access_token`
- A date stored as UTC is returned without timezone annotation
- An async function is called without `await` at the call site

---

## What to avoid

DO NOT generate code based on what you think the schema probably is. Read the schema. Ask if not provided.

DO NOT use `as any` to make a type error go away. Fix the underlying type.

DO NOT expand scope without explicit approval. Unsolicited changes are not improvements.

DO NOT claim anything is "done" until the output has been verified to be internally consistent.

DO NOT invent library APIs. If you are not certain a method exists, say so.

DO NOT silently choose between two approaches without surfacing the decision.
State: "I chose [A] over [B] because [reason]."

DO NOT repeat an attempt that already failed without changing your approach.
If the first attempt did not work, explain why and take a different path.

---

## Checklist for AI Agent Responses

Before every code response, confirm internally:

- [ ] I have read -- not assumed -- the content of every file I am modifying
- [ ] All import paths I used exist in the provided codebase context
- [ ] All functions I call are defined in the provided context
- [ ] My scope statement matches what I am about to generate
- [ ] I have not added anything outside the stated scope
- [ ] Where I am uncertain, I have flagged it explicitly with UNCERTAIN or BLOCKED
- [ ] No real secrets, API keys, or user IDs appear in my output
- [ ] All placeholders are marked // PLACEHOLDER
- [ ] Types are consistent across all files in this response
- [ ] Function call sites match function signatures
- [ ] My response format matches what was specified in the instruction
- [ ] I have stated what was completed and what requires human verification
