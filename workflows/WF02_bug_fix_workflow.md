# WF02 -- Bug Fix Workflow

## Purpose

Step-by-step runbook for investigating and fixing any bug. Fixes produced without
investigation are guesses. Guesses often fix the symptom and leave the root cause.
This workflow forces root-cause diagnosis before any code change.

---

## Pre-conditions

- You have a bug report: what happened, what was expected
- You can reproduce the issue (or the issue is reproducible in a specific environment)

If you cannot reproduce the bug, do not attempt a fix. Begin with Step 1.

---

## Step-by-step

### Step 1: Reproduce the bug

Before investigating, confirm you can make the bug happen.

Where to reproduce:
1. Locally -- fastest feedback loop
2. Staging -- if the bug only appears with real data
3. Identify exact conditions that trigger the bug:
   - What user action or API call triggers it?
   - What data state is required?
   - Is it 100% reproducible or intermittent?

If intermittent: look for race conditions, timing dependencies, or data-conditional paths.

Do not proceed until you can reliably trigger the bug.

---

### Step 2: Read the error

If there is an error message, read the full stack trace. Not just the first line.

Checklist:
- [ ] What is the error type? (TypeError, DatabaseError, ValidationError?)
- [ ] What is the exact error message?
- [ ] What file and line number is in the stack trace?
- [ ] Is this a user-facing error or a server-side error?
- [ ] Check the logs for the request timeline -- what happened before the error?

---

### Step 3: Locate the failing code

Do not guess where the bug is. Trace it:

1. Start from the error location in the stack trace
2. Follow the call chain backwards to the entry point
3. Find the exact line where the incorrect behavior originates

Tools:
- Add `console.log` or a debugger breakpoint at each level of the call chain
- Run only the isolated function in a test to narrow scope
- Read the database state before and after the failing operation

---

### Step 4: Form a hypothesis

Before changing any code, write down ONE sentence:
```
I believe the bug is caused by [specific thing] in [specific location].
```

Then write the test that would confirm or deny this hypothesis -- before writing the fix.

---

### Step 5: Write a failing test

Write a test that:
1. Reproduces the exact bug
2. Currently fails (confirming the bug is real)
3. Will pass once the fix is in place

This test is not optional. A bug without a regression test will come back.

```typescript
it('should [expected behavior] when [condition that was failing]', async () => {
  // Arrange: set up the exact conditions that trigger the bug
  // Act: trigger the failing operation
  // Assert: verify the correct behavior
});
```

Run the test. Confirm it fails.

---

### Step 6: Fix the root cause

Now write the fix. Target only the root cause. Do not "clean up" anything else.

Rules:
- Fix the minimum amount of code required to resolve the root cause
- Do not refactor surrounding code unless the refactor is directly required
- If fixing one thing reveals another bug, file a separate ticket -- do not fix two bugs in one PR
- If the fix requires a data migration (corrupt data to repair), document it separately

---

### Step 7: Run the regression test

Run the test you wrote in Step 5. It must now pass.

Then run the full test suite:
```bash
pnpm test
```

If any previously passing test fails, your fix introduced a regression. Investigate
why the test broke before proceeding.

---

### Step 8: Verify end-to-end

Reproduce the original bug scenario manually. Confirm the behavior is correct now.

Then verify adjacent behavior:
- Did the fix affect any related functionality?
- Are there other code paths that had the same bug pattern?

---

### Wrap-up

```bash
tsc --noEmit
eslint .
pnpm test
```

PR description for bug fixes must include:

```markdown
## Bug

[What was happening and in what conditions]

## Root Cause

[The actual cause -- not the symptom]

## Fix

[What was changed and why this fixes the root cause]

## Regression Test

[Name of the test that was added to prevent recurrence]

## How to Verify

[Steps to manually confirm the fix]
```

---

## Done definition

A bug fix is not done until:
- A failing regression test exists and now passes
- Full test suite passes
- The original bug scenario is manually verified as fixed
- PR is reviewed and merged
- If the bug was in production: confirm the fix is deployed and the issue does not recur
