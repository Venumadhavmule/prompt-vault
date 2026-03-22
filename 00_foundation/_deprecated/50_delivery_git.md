# SKILL 5.1 -- How to Use Git and Version Control Like a Senior Engineer

## What this skill is

This skill covers the Git workflow, commit discipline, PR practices, and version
control hygiene that make codebases maintainable over years. Bad Git practices result
in lost work, merge nightmares, accidental secret exposure, and a history that is
useless as documentation. Following this skill produces a Git log that tells the
story of the software and a workflow that enables fast, safe collaboration.

---

## When to use this skill

- When starting any work on a codebase
- When writing commit messages
- When opening a pull request
- When reviewing how code history is organized
- When recovering from a Git mistake

---

## Full Guide

### Step 1: Branch naming

Every branch has a clear type and a descriptive slug.

| Purpose | Pattern | Example |
|---|---|---|
| Feature | `feature/<ticket>-<slug>` | `feature/PV-42-add-stripe-webhooks` |
| Bug fix | `fix/<ticket>-<slug>` | `fix/PV-88-token-expiry-race` |
| Hotfix (prod) | `hotfix/<slug>` | `hotfix/payment-double-charge` |
| Release | `release/<version>` | `release/1.3.0` |
| Chore / tooling | `chore/<slug>` | `chore/upgrade-prisma-5` |
| Experiment | `spike/<slug>` | `spike/websocket-scaling` |

Branch rules:
- Branch from `main` (or your default branch) always
- Never commit directly to `main` -- all changes go through PRs
- Delete the branch after merging
- Keep branches short-lived: days, not weeks. Long-lived branches = merge hell.

---

### Step 2: Commit message format (Conventional Commits)

```
<type>[optional scope][optional !]: <summary>

<body>
```

**Types:**
- `feat`: new feature (MINOR version bump)
- `fix`: bug patch (PATCH version bump)
- `refactor`: restructure, no behavior change
- `chore`: tooling, dependencies, config
- `update`: enhance existing feature
- `docs`: documentation only
- `build`: build system or CI changes
- `test`: adding or fixing tests
- `perf`: performance improvement
- Breaking changes: append `!` after type

**Rules:**
- Summary: imperative mood, present tense ("Add" not "Added")
- Summary: maximum 72 characters
- Summary: do not end with a period
- Body: explain WHAT changed and WHY (not HOW -- the code shows how)
- One blank line between summary and body

**Examples:**
```
feat(auth): Add refresh token rotation with reuse detection

Implement automatic refresh token rotation on each refresh call.
Store token family ID to detect reuse. When reuse is detected,
revoke all tokens in the family and force re-authentication.
Prevents token theft replay attacks.

fix(payments): Correct proration calculation for mid-cycle upgrades

Was using subscription start date instead of current period start.
This caused incorrect charge amounts when upgrading mid-cycle.
Now uses Stripe's proration_date parameter set to current timestamp.

refactor(db): Extract query builder into shared utility module

Payment, user, and subscription repositories each had duplicate
query building logic. Extracted into packages/db/query-builder.ts
to eliminate the duplication and make the logic testable in isolation.
```

---

### Step 3: Atomic commits

One commit = one logical change. The commit message should completely describe it.

**Signs of a bad commit:**
- "Fix stuff" -- no context
- "WIP" -- work in progress should not be in main history (squash before merging)
- One commit that changes 20 files across 5 different concerns
- A commit where you have to read diff to understand what it was trying to do

**Signs of a good commit:**
- Adding one feature: the diff and message together tell the complete story
- Fixing one bug: the test that was failing + the fix that makes it pass
- Refactoring one thing: before and after are structurally clear

---

### Step 4: The branch workflow

```
1. Create branch from main
   git checkout main && git pull origin main
   git checkout -b feature/PV-42-add-webhook-endpoint

2. Work in small, atomic commits as you go
   (do not save all commits for the end)

3. Before pushing: self-review
   git diff origin/main...HEAD -- look at every change
   Run lint, type-check, and tests

4. Push and open PR
   git push origin feature/PV-42-add-webhook-endpoint
   Open PR against main (or staging if you have that branch)

5. Address review feedback in new commits
   (do not amend published commits in an open PR -- confusing for reviewers)

6. Squash/clean up before merge
   git rebase -i origin/main (interactive rebase to clean history)
   -- squash "fix typo" and "address review" commits
   -- keep meaningful separate commits if PR has multiple logical changes

7. Merge via PR (squash merge for single-concern PRs, merge commit for multi-concern)

8. Delete branch
```

---

### Step 5: Pull Request description format

A PR description is documentation. Reviewers need context, not just a diff.

```markdown
## What

[1-2 sentences: what does this PR do?]

## Why

[1-2 sentences: why is this change needed? What problem does it solve?]

## How

[Optional: if the approach is non-obvious, explain the key decisions]

## Testing

[What tests were added or changed? how to verify manually if applicable]

## Screenshots

[For UI changes: before and after screenshots]

## Checklist

- [ ] Tests pass locally
- [ ] TypeScript compiles without errors
- [ ] Migration included (if schema changed)
- [ ] .env.example updated (if new env var added)
- [ ] No secrets committed
- [ ] Self-reviewed the diff
```

---

### Step 6: Handling merge conflicts

Merge conflicts are normal. Handle them systematically:

1. Fetch the latest main: `git fetch origin`
2. Rebase your branch: `git rebase origin/main`
3. For each conflicted file: read BOTH versions and understand the intent
4. Resolve: integrate both changes if both are needed, or keep one if the other is superseded
5. Test after resolving -- conflicts often introduce subtle bugs
6. Never "choose one side" blindly without understanding what you are discarding

---

### Step 7: When to squash vs preserve commits

**Squash into one commit:**
- A small feature or bug fix that is conceptually one change
- A PR with "fix typo", "address feedback", "try again" commits
- When the intermediate commits add no historical value

**Preserve multiple commits:**
- A PR with multiple independent logical changes (each tells a different story)
- A complex feature where the sequence of refactor → feature → tests tells the story
- Migrations + application code together (each as a separate commit)

---

### Step 8: Safely undoing things in Git

| Situation | Command | Safety |
|---|---|---|
| Undo last commit, keep changes staged | `git reset --soft HEAD~1` | Safe |
| Undo last commit, keep changes unstaged | `git reset HEAD~1` | Safe |
| Undo last commit, discard changes | `git reset --hard HEAD~1` | Local only -- dangerous if pushed |
| Undo a specific commit in history | `git revert <hash>` | Safe for shared branches (creates a new commit) |
| Remove a file from staging | `git reset HEAD <file>` | Safe |
| Discard all local changes | `git checkout .` | Destructive -- all uncommitted changes lost |
| Recover a deleted branch | `git reflog` to find hash, then `git checkout -b name <hash>` | Works until GC |

**Golden rule for shared branches (main, staging):** ALWAYS use `git revert`, never `git reset --hard`.
`git reset --hard` on a shared branch rewrites history that others have already pulled. This causes chaos.

---

### Git hygiene: what never to commit

- `.env` files with real values (use `.gitignore` and `.env.example`)
- `node_modules/` (should always be in `.gitignore`)
- Build artifacts: `dist/`, `.next/`, `build/`
- IDE files: `.vscode/`, `.idea/` (unless you explicitly want to share settings)
- API keys, passwords, JWT secrets, private SSH keys
- Large binary files (use Git LFS if large files must be tracked)

If a secret is accidentally committed, the correct response is:
1. Immediately revoke/rotate the exposed credential (before anything else)
2. Remove from repository history using `git filter-repo`
3. Force push to all branches
4. Audit all logs to assess if the secret was accessed

---

## What to avoid

DO NOT commit directly to `main`.

DO NOT use "WIP" or "fix stuff" as commit messages.

DO NOT use `git reset --hard` on shared branches.

DO NOT leave secrets in any commit, even "just for now" -- Git history is permanent.

DO NOT let a branch live for more than a week without merging or rebasing on main.

DO NOT squash all commits into one when the PR has multiple distinct changes worth preserving.

---

## Checklist

Before pushing any branch:

- [ ] Branch name follows the naming convention
- [ ] Each commit has a clear Conventional Commit message (type + summary + body)
- [ ] No "WIP", "fix stuff", or "try again" commits in the history
- [ ] Self-review: run `git diff origin/main...HEAD` and read every change
- [ ] Lint passes, TypeScript compiles, tests pass
- [ ] No `.env` files or secrets anywhere in the diff
- [ ] `node_modules/` and build artifacts are in `.gitignore`
- [ ] PR description follows the template (what, why, how, testing, checklist)
- [ ] Branch is up to date with main (rebased or merged)
- [ ] Build artifacts and IDE files are NOT in the commit
