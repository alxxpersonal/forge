---
name: commit-forge
description: Write clean, atomic git commits with conventional format. Use when committing code, staging changes, or user says "commit", "commit this", "stage and commit", or is done with a task and needs to commit. NOT for git troubleshooting or branch management.
---

# Commit Forge

Atomic, descriptive git commits. Every commit is one logical change that leaves the codebase working.

## Hard Rules

- **NEVER** commit without the user explicitly asking
- **NEVER** add co-author tags to commits
- **NEVER** skip hooks (`--no-verify`, `--no-gpg-sign`)
- **NEVER** force push to main/master
- **NEVER** use `git add .` or `git add -A` - stage specific files by name
- **PREFER** creating a new commit over amending an existing one

## Commit Message Format

```
type(scope): short description (max 72 chars)

Optional body - what and why, not how.

Optional footer - breaking changes, issue refs.
```

### Types

| Type | Use For |
|------|---------|
| `feat` | New feature or capability |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `refactor` | Code restructuring, no behavior change |
| `test` | Adding or fixing tests |
| `chore` | Maintenance, deps, config, CI |
| `style` | Formatting, no logic change |
| `perf` | Performance improvement |

### Scope

Use short, lowercase scope matching the area of change. Read the repo's existing commits (`git log --oneline -20`) to match the project's conventions.

### Examples

```
feat(auth): add JWT refresh token rotation

fix(parser): handle empty broker email attachments

refactor(cli): extract table renderer to shared package

chore: update go dependencies

test(api): add integration tests for driver matching
```

## Process

### Step 1: Review Changes

```bash
git status
git diff
git diff --staged
```

Understand what changed before staging anything.

### Step 2: Stage Selectively

Stage files that belong to one logical change:

```bash
git add src/auth/handler.go
git add src/auth/handler_test.go
```

If the diff contains multiple unrelated changes, split into separate commits.

### Step 3: Verify Before Commit

Auto-detect and run the repo's checks:

| Indicator | Command |
|-----------|---------|
| `Makefile` with test target | `make test` |
| `package.json` with test script | `npm test` / `pnpm test` |
| `go.mod` exists | `go test ./...` |
| `Cargo.toml` exists | `cargo test` |
| `pyproject.toml` / `setup.py` | `pytest` |

Only run if tests exist and are fast. Skip if the user explicitly says to commit without testing.

### Step 4: Commit

Write a message that answers "what changed and why" - not "how."

```bash
git commit -m "feat(auth): add rate limiting to login endpoint"
```

For complex changes, use a body:

```bash
git commit -m "refactor(api): split monolithic handler into middleware chain

Request validation, auth, and response formatting were tangled
in a single 400-line handler. Split into composable middleware
for independent testing and reuse."
```

### Step 5: Verify

```bash
git log --oneline -3
git show --stat
```

Confirm the commit looks right.

## What Makes a Good Commit

| Good | Bad |
|------|-----|
| One logical change | Multiple unrelated changes |
| All tests pass | Breaks something |
| Message says why | Message says "update stuff" |
| Can be reverted cleanly | Reverting would break other things |
| Specific files staged | `git add .` |

## What Makes a Bad Commit Message

- "fix stuff" / "update" / "changes" / "WIP"
- Using "and" to describe two unrelated things
- Describing how instead of what/why
- Over 72 characters in the subject line

## Checklist

Before committing:

- [ ] Changes are atomic (one logical unit)
- [ ] Specific files staged (not `git add .`)
- [ ] Tests pass (if applicable)
- [ ] Message follows `type(scope): description` format
- [ ] No co-author tags
- [ ] No sensitive files staged (.env, credentials, keys)
