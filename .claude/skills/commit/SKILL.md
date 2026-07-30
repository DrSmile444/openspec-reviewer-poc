---
name: commit
description: Creates Conventional Commits with automatic task name extraction from branch names. Use whenever any git commit is needed — whether the user asks to commit or Claude decides to commit as part of completing a task.
allowed-tools: Bash(git:*), Read, Grep, Glob
metadata:
  author: "Dmytro Vakulenko"
  version: "1.1.0"
---

# Commit Skill

You are creating git commits following the **Conventional Commits** specification with automatic task name extraction from the current branch.

## Step 1 — Determine the task name

Run `git branch --show-current` to get the current branch name.

Extract the task identifier using this pattern: `[A-Z]+-\d+` (e.g., `TASK-123`, `UABOT-240`, `PROJ-42`).

Examples:
- `feature/UABOT-240-fix-roles-errors` → task = `UABOT-240`
- `fix/PROJ-42-something` → task = `PROJ-42`
- `main`, `develop`, `master`, or any branch without a matching pattern → no task

## Step 2 — Inspect the changes

Run the following to understand what has changed:

```bash
git status
git diff
git diff --cached
```

Group the changes by logical concern (e.g., one group for config changes, another for feature code, another for tests). Each logical group becomes a separate commit.

## Step 3 — Commit iteratively

**Do not make one big commit.** Split the changes into small, focused commits — one per logical concern. This makes history easier to review.

For each group of changes:

1. Stage only the relevant files with `git add <files>`.
2. Write a commit message following the format below.
3. Run `git commit -m "..."`.
4. Move on to the next group.

Never create an empty commit. If there are no changes to commit, report that to the user and stop.

## Commit message format

### With task name (branch has a task identifier):

```
<type>(<TASK-ID>): <short description>
```

Examples:
```
feat(UABOT-240): add video processing support
fix(UABOT-240): resolve role assignment error
chore(UABOT-240): update dependencies
```

### Without task name (main, develop, or no task in branch):

```
<type>: <short description>
```

Examples:
```
feat: add video processing support
fix: resolve role assignment error
chore: update dependencies
```

## Conventional Commits type reference

| Type       | When to use                                                   |
| ---------- | ------------------------------------------------------------- |
| `feat`     | A new feature                                                 |
| `fix`      | A bug fix                                                     |
| `chore`    | Maintenance tasks, dependency updates, tooling, config        |
| `refactor` | Code restructuring without behavior change                    |
| `test`     | Adding or updating tests                                      |
| `docs`     | Documentation only changes                                    |
| `style`    | Formatting, missing semicolons — no logic change              |
| `perf`     | Performance improvement                                       |
| `ci`       | CI/CD pipeline changes                                        |
| `build`    | Build system or external dependency changes                   |
| `revert`   | Reverts a previous commit                                     |

## Rules

- Keep the subject line under 100 characters.
- Use imperative mood: "add feature" not "added feature".
- Do not end the subject line with a period.
- One logical concern per commit — never bundle unrelated changes.
- Never skip hooks (`--no-verify`) unless the user explicitly asks.
- Never create empty commits.
- After all commits are done, run `git log --oneline -10` and show the result so the user can review the commit history.
