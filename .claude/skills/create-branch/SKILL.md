---
name: create-branch
description: Creates a new git branch named per the Conventional Branch spec (conventionalbranch.org) — <type>/<TASK-ID>-<short-desc> — branching either from the current branch (pull first) or fresh from the remote trunk (fetch first). Automatically reuses the Jira/task ID you're already working on (e.g. UA-100) as the branch scope. Use this whenever the user wants to start a new branch or begin new work — phrases like "create a branch", "new branch", "branch for this", "start a branch off main", "branch off the current one", "spin up a branch for UA-100", or when they begin a fresh task/ticket and need somewhere to work. Trigger even if they don't say the word "conventional" or name the format.
allowed-tools: Bash(git:*), Read, Grep, Glob
metadata:
  author: "Dmytro Vakulenko"
  version: "1.0.0"
---

# Create Branch Skill

You are creating a new git branch that follows the **Conventional Branch** specification (conventionalbranch.org, v1.1.0). A good branch name is purpose-driven and machine-readable: the prefix says *what kind of work* this is, the scope ties it to a *ticket*, and the description says *what* in a few words. That naming also lets the downstream `commit` skill extract the task ID automatically, so the two compose — get the name right here and commits/PRs inherit it for free.

Work through the four decisions in Step 1 first (no git side effects yet), assemble the name, **show a preview and wait for approval**, then create the branch.

## Step 1 — Decide four things before touching git

Resolve these from the request and the existing conversation. Do **not** start querying Jira or running mutating git commands to figure them out — use what's already in front of you.

### 1a. Base source — current branch, or remote trunk?

Two paths, and the user's wording almost always tells you which:

- **Remote trunk** (fetch first, branch off fresh `main`/`master`): cues like "off main", "from master", "fresh branch", "new branch from trunk", "based on the latest main", or when they're clearly starting unrelated new work.
- **Current branch** (pull first, branch off where they are): cues like "from here", "off this branch", "stack on top of this", "continue from this one", "branch off the current one". This is the right call when they're building on work that isn't merged yet (stacked branches).

If neither is implied, **ask one short question** ("Branch off the current branch (`<name>`) or fresh from remote `<trunk>`?") rather than guessing — the base determines everything downstream.

### 1b. Type prefix

Infer the prefix from the nature of the work, and surface it in the preview so the user can override:

| Prefix | When |
| --- | --- |
| `feature/` | New functionality, enhancements (the default when unsure) |
| `bugfix/` | Fixing a non-urgent bug |
| `hotfix/` | Urgent production fix that can't wait for the normal flow |
| `chore/` | Dependencies, tooling, config, docs, non-code maintenance |
| `release/` | Preparing a release (description is usually a version, e.g. `release/v1.2.0`) |

Use the **full word** form (`feature`, not `feat`; `bugfix`, not `fix`) — that's the house style. If the user names a type explicitly, honour it.

### 1c. Task ID (the scope)

The scope is the Jira/ticket key, kept **UPPERCASE** (e.g. `UA-100`, `UABOT-240`, `PROJ-42`), matching the pattern `[A-Z]+-\d+`. Find it in this priority order:

1. **Explicit in the request** — "branch for UA-100 …".
2. **The ticket in play in this conversation** — e.g. a Jira issue you just explored/discussed. If the user explored `UA-100` and now says "create a branch", that's the scope.
3. **The current branch name**, *only* when branching off it (1a = current) and it already carries a task ID — reuse it, but flag it in the preview since a sub-task may have its own key.

If no task ID is found, **omit the scope** and build `<type>/<description>`. Note "no task ID detected" in the preview so the user can supply one if they meant to.

### 1d. Description slug

Turn the purpose into a short, meaningful kebab-case slug:

- Lowercase; words separated by single hyphens.
- **Drop filler** — articles, prepositions, and connectors (`a`, `the`, `to`, `for`, `of`, `and`, `in`, `on`, `with`, `that`, `just`, `some`) that carry no meaning. Keep the words that identify the work.
- Strip anything that isn't `a-z`, `0-9`, or `-` (dots only in `release/` version strings).
- **7 words max** — aim for 3–5. Concise beats complete; the ticket holds the full context.

Example: "improve the performance of the SQL query for reports" → `improve-sql-query-performance`.

## Step 2 — Assemble the branch name

```
<type>/<TASK-ID>-<description>      # with a task ID
<type>/<description>                # without
```

Examples:
- `feature/UA-100-improve-sql-performance`
- `bugfix/UABOT-240-fix-role-assignment`
- `hotfix/restore-checkout-flow`  *(no ticket)*
- `release/v1.2.0`

Validate against the rules in **Branch naming rules** below before showing it.

## Step 3 — Preview and confirm

Always show, and wait for the user's OK (they can tweak any part):

```
Base:    remote trunk (origin/main)        ← or: current branch (feature/UA-99-...)
Branch:  feature/UA-100-improve-sql-performance
Steps:
  git fetch origin main
  git checkout -b feature/UA-100-improve-sql-performance origin/main
```

Call out anything notable here: no task ID detected, a dirty working tree, or a reused task ID from the current branch.

## Step 4 — Create the branch

Pre-flight (both paths): confirm you're in a repo (`git rev-parse --is-inside-work-tree`) and that the name is free (`git show-ref --verify --quiet refs/heads/<name>` → if it exists, stop and ask for a different name).

### Path A — from the current branch (pull first)

```bash
git status --porcelain          # check the working tree
git pull --ff-only              # update the current branch
git checkout -b <name>
```

- **Dirty working tree:** `git pull --ff-only` will refuse if changes conflict. Don't force or silently stash — report it and let the user decide (stash, commit, or abort).
- **No upstream / diverged:** if `--ff-only` fails because there's no tracking branch or the branch has diverged, stop and report rather than doing a merge or rebase behind their back.

### Path B — from the remote trunk (fetch first)

```bash
git fetch origin <trunk>
git checkout -b <name> origin/<trunk>
```

Branching directly off `origin/<trunk>` means you don't have to checkout or pull local trunk first — the new branch starts from the freshly-fetched remote tip.

**Detecting the trunk** (don't hardcode `main` — repos differ):

```bash
git symbolic-ref --short refs/remotes/origin/HEAD     # → origin/main
```

If that errors (origin/HEAD not set locally), fall back to asking the remote directly:

```bash
git ls-remote --symref origin HEAD                    # → ref: refs/heads/main  HEAD
```

Uncommitted changes carry over to the new branch on `checkout -b`, which is usually fine — mention it in the preview if the tree is dirty so it's not a surprise.

## Branch naming rules (Conventional Branch 1.1.0)

- Format is `<type>/<description>`; trunk branches (`main`, `master`, `develop`) have no prefix.
- Description uses lowercase `a-z`, `0-9`, and `-`. Dots `.` only in `release/` versions.
- No spaces, underscores, or other special characters.
- No consecutive hyphens/dots (`new--login` ✗), and none at the start or end of the description (`-new-login` ✗, `new-login-` ✗).
- **Exception (house rule):** the task ID keeps its native UPPERCASE form (`feature/UA-100-…`). This diverges from the spec's strict-lowercase grammar on purpose — it matches Jira's own keys and lets the `commit` skill grep the ID back out.

## Don'ts

- Don't push or set upstream — leave the branch local. (The branch may be renamed before its first push, and pushing is the PR step's job.)
- Don't run `git fetch`/`git pull`/`checkout -b` before the user approves the preview.
- Don't query Jira just to find a task ID — use the one already in the conversation, or proceed without a scope.
- Don't hardcode `main`; detect the trunk.
- Don't pad the description with filler or exceed 7 words.
