# SKILL 3.6 -- How to Debug Any Problem Systematically

## What this skill is

This skill teaches a debugging process that works for any type of problem in any
stack. Debugging without a system leads to random changes, wasted time, and fixes
that address symptoms rather than root causes. This skill eliminates debugging by
instinct and replaces it with a scientific process: form a hypothesis, test it,
and fix the root cause -- not the surface.

This skill applies to AI coding agents just as much as humans. An agent that blindly
retries different code changes without diagnosing the root cause will produce broken
output faster than a good diagnostic process.

---

## When to use this skill

- When a test is failing and you do not immediately know why
- When production has an incident and you are investigating
- When a bug is reported and you need to find the root cause
- When code behaves differently than expected
- When an AI agent receives a failing output and needs to course-correct

---

## Full Guide

### The debugging mindset

You are a scientist. Not a guesser.

The process is:
1. Observe (what is the actual behavior?)
2. Hypothesize (what is the most likely cause?)
3. Test the hypothesis (prove or disprove with a targeted experiment)
4. Draw a conclusion (is the hypothesis correct? what do you now know?)
5. Fix the root cause (not the symptom)
6. Prove the fix (the test that previously failed must now pass)

NEVER:
- Make a change and hope it works
- Comment out code and see if the problem disappears without understanding why
- Copy a Stack Overflow answer without knowing what it does
- Make multiple changes at once (you will not know which one fixed it)

---

### Step 1: Reproduce reliably

You cannot fix what you cannot reproduce.

**For code bugs:**
- Write a failing test that demonstrates the bug. This becomes your regression test automatically.
- If you cannot write a test: reproduce it with the exact same input, environment, and state.

**For environment-specific bugs:**
- Find the minimal reproduction: "Does it happen in a test? In staging? Only in prod?"
- Capture: exact error message, full stack trace, environment variables in effect, input data

**If you cannot reproduce reliably:**
- Add logging at the suspected entry points
- Deploy the logging change and wait for the bug to appear again
- Do NOT attempt a fix before reproducing. You will fix the wrong thing.

---

### Step 2: Read the actual error

Read the FULL error message and the COMPLETE stack trace. Not just the last line.

**What the stack trace tells you:**
- The exact line where the error was thrown
- Every function in the call stack leading to that line
- The first function YOU wrote (not a library) from the top -- that is usually where the bug is

**Common mistake:** Googling the error message without looking at the line where it occurred in YOUR code.

**For async errors:** The stack trace in async code can be misleading. Node.js async operations
lose context. Use `--async-stack-traces` or async stack trace tools in your environment.

**For database errors:** The DB error message contains the actual constraint or syntax that failed.
Read the full error, not just "DB query failed."

---

### Step 3: Isolate to the smallest failing unit

Use binary search to narrow down the problem:

1. Is the bug in the frontend or backend? (test the API directly)
2. Is it in the service layer or the repository? (test the service with mocked repository)
3. Is it in a specific function or in how it's called? (test the function in isolation)

**Binary search technique:**
Add a log or assertion at the midpoint of the execution path.
If it passes, the bug is in the second half. If it fails, the bug is in the first half.
Repeat until you've isolated to a single function or even a single line.

---

### Step 4: Form a hypothesis

State your hypothesis in one sentence:
"I believe the bug is caused by [specific cause] because [evidence]."

Examples:
- "I believe the bug is that `findById` returns `undefined` instead of `null` when the user does not exist, because the type definition says `null` but Prisma returns `undefined` for missing records."
- "I believe the token expiry check is using server time but the JWT was issued with client clock time, and there is a 5-minute clock skew."
- "I believe the N+1 is caused by the ORM eager-loading not being applied to the latest version of the query after the schema change."

If you cannot state a hypothesis, you have not isolated far enough. Go back to Step 2.

---

### Step 5: Test the hypothesis with a targeted change

Test one hypothesis at a time. Add a targeted log or assertion that proves or disproves it.

```typescript
// Hypothesis: findById returns undefined not null
const user = await userRepository.findById(id)
console.log('DEBUG findById result:', user, typeof user)
// → confirms undefined vs null
```

Make ONE change to test the hypothesis. Observe the result. Either:
- The hypothesis is CONFIRMED → proceed to fix
- The hypothesis is WRONG → go back to Step 4 with new information

NEVER make multiple changes simultaneously to "try things." That destroys your ability to learn
from the experiment.

---

### Step 6: Fix the root cause, not the symptom

Symptoms are what you observe. Root causes are what caused the symptom.

**Symptom:** User is getting logged out every 15 minutes
**Symptom fix (wrong):** Extend the access token expiry to 7 days
**Root cause:** Refresh token rotation is failing silently due to a race condition
**Root cause fix (right):** Fix the race condition in the refresh token handler

**Symptom:** Stripe webhook is processing duplicate events
**Symptom fix (wrong):** Ignore if subscription is already active
**Root cause:** Idempotency key is not being set on the webhook handler
**Root cause fix (right):** Add idempotency check against processed event IDs

Ask "why" 5 times before accepting your fix as the root cause:
- Why is the user logged out? → Access token expired
- Why did it expire? → Refresh failed
- Why did refresh fail? → Token was missing from DB
- Why was it missing? → Token was deleted in a cleanup job
- Why was it deleted? → Cleanup job uses wrong condition (deletes unrotated tokens)

The root cause is at the bottom of the "why" chain.

---

### Step 7: Write the regression test first

BEFORE implementing the fix, write the test that proves the fix is needed and observes the failure.

1. Write the test describing the expected behavior
2. Run it -- it must fail
3. Implement the fix
4. Run the test -- it must pass
5. Run the full test suite -- no other tests must break

This approach guarantees:
- You understand what "fixed" means before you fix it
- The regression never ships again
- Your fix actually addresses the stated behavior

---

### Debugging specific problem types

**Async / timing issues:**
- Use `await` consistently -- one missing `await` is often the culprit
- Add timestamp logs to find sequences that execute out of expected order
- Race conditions: test with concurrent requests (use Promise.all with multiple parallel calls)
- Timers/intervals: verify cleanup in useEffect or component teardown

**Database query issues:**
- Use EXPLAIN ANALYZE to see what the query planner is doing
- Log the actual SQL being generated (Prisma: set `log: ['query']` in dev)
- Check for N+1: how many queries ran for one request? (enable query count logging)
- Check for missing indexes: EXPLAIN ANALYZE shows "Seq Scan" when an index should be used

**Authentication issues:**
- Decode the JWT manually: `atob(token.split('.')[1])` -- is the payload correct?
- Check `iat` and `exp` in the decoded JWT against current server time
- Verify the JWT secret matches between the issuer and verifier
- Check if the token is being sent in the correct header format (`Bearer [token]`)
- Verify CORS is not stripping the Authorization header

**Production-only issues:**
- Compare env vars between staging and production
- Check for data that exists only in production (edge cases in real user data)
- Check for timing in production load that does not exist under dev load
- Add structured logs with a requestId, then search the log system for that ID

---

### The debugging checklist (10 things to check before going deeper)

1. Is the error exactly what you think it is? Read the full stack trace.
2. Does the bug reproduce locally or only in a specific environment?
3. Has anything changed recently? (last git commit, deploy, dependency update, data change)
4. Are you looking at the right logs? (correct service, correct time range, correct log level)
5. Is the input exactly what you expect? (log the input, do not assume)
6. Is a dependency returning null/undefined when you expect a value?
7. Are all async operations awaited?
8. Is the error thrown in your code or in a library? If library, check your usage.
9. Is the same pattern used elsewhere without bugs? What is different here?
10. Is there a test that covers this exact path? If not -- write it now, it will tell you what's wrong.

---

## What to avoid

DO NOT make random changes hoping one will fix the bug.

DO NOT fix a symptom without investigating the root cause.

DO NOT make multiple changes simultaneously -- you will not learn what worked.

DO NOT copy code from the internet without understanding what it does.

DO NOT close a bug as fixed without a regression test.

DO NOT assume the bug is in the "obvious" place. Always verify your assumption first.

DO NOT debug asynchronous code by adding synchronous delays/sleep -- fix the actual async handling.

---

## Checklist

For any bug investigation:

- [ ] I have reproduced the bug reliably (or written a failing test)
- [ ] I have read the full stack trace, not just the last line
- [ ] I have identified the first function in MY code in the stack trace
- [ ] I have isolated the bug to the smallest failing unit
- [ ] I have formed a hypothesis in one sentence
- [ ] I have tested the hypothesis with one targeted change
- [ ] I have confirmed the hypothesis before fixing
- [ ] My fix addresses the root cause, not the symptom
- [ ] I have verified "why" at least 3 times to confirm root cause
- [ ] A regression test has been written that demonstrates the fix
- [ ] The full test suite passes after the fix
- [ ] Debug logs added during investigation have been removed
- [ ] The fix has been communicated: what the root cause was, how it was fixed
