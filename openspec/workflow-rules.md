# Workflow rules payload

This file is the single source of truth for the portable part of the workflow: the exact
`rules.tasks` and `rules.proposal` list items that get **merged into** a target project's
`openspec/config.yaml`, as described in the "Installation via agent" section of
[`README.md`](../README.md).

Nothing else in `config.yaml` — `schema`, `context`, `external_docs`, or any other artifact's
rules (`design`, `specs`, `adrs`, `plan`, `self_review`, …) — is part of this payload. Those
stay project-specific and an install must never touch them.

This repo's own [`config.yaml`](config.yaml) embeds the same `rules.tasks`
content shown below. Keep the two in sync when the workflow changes; this file exists so an
install prompt has an unambiguous block to read and append, without having to extract it out
of a config file whose other fields differ from project to project.

## Version marker

The first and last lines of the `rules.tasks` payload below are start/end markers:

```
openspec-reviewer-poc workflow v1 — branch, implement, verify, commit, review, fix, commit.
...
openspec-reviewer-poc workflow v1 — end of managed block. Rules after this line, if any,
are not managed by this install and must never be touched by an upgrade.
```

The end marker is what makes an upgrade safe: the *managed block* is every `rules.tasks`
item from the start marker to the end marker carrying the same version number, inclusive —
nothing before or after it, no matter what the target project appended around it. A version
marker with no matching end marker (a start marker is present but nothing after it in the
list matches `openspec-reviewer-poc workflow v<same-N> — end`) is a shape the install prompt
does not know how to handle - it stops and asks, rather than guessing where the block ends.

An install checks the target's existing `rules.tasks` for the substring
`openspec-reviewer-poc workflow v` before doing anything:

- **no start marker found** → append this whole block (start marker through end marker) to
  `rules.tasks`.
- **start marker found, same version** → compare the target's managed block item-by-item
  against this one. Identical → already installed, no changes. Different (an item edited,
  or the end marker missing/moved) → show what differs and ask before repairing it back to
  this reference block, rather than silently reporting "already installed" over a partial or
  hand-edited install.
- **start marker found, older version** → show the target's existing managed block and this
  one, and ask before replacing the old block with the new one - replacing only the exact
  item range between the two markers, never anything outside it.

Independently of which of the three cases above applies, the install also checks whether
`rules.proposal` already has a rule capturing a `Task: <ID>` line and appends the bullet
below if not - this check is not gated by the `rules.tasks` version marker.

Before any of this, the install confirms `rules.tasks` and `rules.proposal` (whichever
already exist in the target) are each a flat list of strings - the shape this payload
assumes. If either exists as something else, it stops and asks rather than writing anything.

Bump this version, and only this version, when the payload's *installed behaviour* changes —
not for unrelated wording polish.

## `rules.proposal` addition

```yaml
- >-
  Record the task tracker ID. If an ID matching [A-Z]+-[0-9]+ is in play - named by the
  user, discussed earlier in the conversation, or present in the current branch name -
  add a line `Task: <ID>` at the end of the Why section. If there is none, write
  `Task: none`. This has to land in the file: later phases run in a fresh context and
  cannot recover it from the conversation.
```

## `rules.tasks` addition

```yaml
- >-
  openspec-reviewer-poc workflow v1 — branch, implement, verify, commit, review, fix,
  commit. The steps this block adds (branch, verify, commit, review, fix, commit) are
  process scaffolding around this project's own implementation tasks, not implementation
  tasks themselves - they are exempt from any rule elsewhere in this list that restricts
  tasks to those derived from a plan, design, or spec document.
- >-
  The rules below are constraints on the task list you generate, not text to copy into
  it. Keep every generated task line to one or two sentences.
- >-
  Task group 1 is `Branch`, one task, resolved before any code is written. A work branch
  is one whose name matches `<feature|bugfix|hotfix|chore>/...`. Anything else - main,
  master, develop, uat, staging, `release/*`, or an unprefixed name - is not a work
  branch, so create one with the create-branch skill. Already on a work branch: reuse it
  if it has zero commits ahead of the trunk and no open PR, or if its task ID matches the
  `Task:` line in proposal.md. Otherwise show the user the branch name, how many commits
  it is ahead of trunk, any open PR, and the new change name, then ask whether to reuse
  it, branch off it, or branch off the trunk. When creating without having asked, default
  the base to the CURRENT branch - stacked branches are expected here; when the user was
  asked, honour whichever base they picked. Use the change name as the description slug.
  If the chosen base has no upstream, skip the `git pull --ff-only` step instead of
  stopping on it.
- >-
  Then the implementation task groups, derived from specs and design.
- >-
  Then a verification task: exercise the change end to end with whatever tooling this
  project and this environment actually offer. Use the `run` skill to drive the app, a
  browser-automation MCP for frontend work, or a throwaway probe script for backend and
  logic. Do not assume a stack or a test runner. If nothing can be exercised, say so
  explicitly rather than ticking the task off in silence.
- >-
  Then a task: commit the implementation with the commit skill. This comes BEFORE the
  reviews on purpose - the built-in /security-review reads `git diff origin/HEAD...`, so
  it only ever sees committed work. Nothing is pushed at this point, so a bad commit is
  still local and rewritable.
- >-
  Then a `Review` group of exactly three tasks. They are independent: launch all three
  together rather than waiting for each to finish, and collect findings only - fix
  nothing yet. The checklist below is flat because that is how tasks.md represents them,
  not because they are sequential.
- >-
  Review (a): /security-review through the Skill tool. Resolve its base first - it reads
  `git diff origin/HEAD...`, so `origin/HEAD` has to exist. If
  `git rev-parse --verify -q origin/HEAD` fails, try `git remote set-head origin -a`. If
  there is no remote at all, this review cannot run: skip it and report "2 of 3 reviews"
  rather than reporting a pass. If it runs and then fails, that is a FAILURE - report it,
  do not silently continue.
- >-
  Review (b): `claude -p '/code-review high'` through Bash. The built-in /code-review is
  marked disable-model-invocation and cannot be called through the Skill tool; a headless
  subprocess is the supported way to run it. It reads branch commits plus the working
  tree, so it works whether or not the implementation is committed.
- >-
  Review (c): Codex adversarial review. Resolve the script first:
  `C=$(ls -d ~/.claude/plugins/cache/openai-codex/codex/*/scripts/codex-companion.mjs 2>/dev/null | sort -V | tail -1)`.
  Empty means the plugin is not installed - skip the task, note it in the review count,
  carry on. Non-empty means run
  `node "$C" adversarial-review --scope branch --base <trunk>`. Use `--scope branch`, not
  `--scope working-tree`: the implementation was committed in the previous step, so the
  working tree is clean and a working-tree review would inspect nothing. A non-zero exit
  is then a review FAILURE, not an absent plugin - do not block on it, but report it as a
  failure rather than filing it under "plugin missing".
- >-
  Then a task: apply fixes for the findings from all three reviews, once every review has
  finished. Not one review at a time - concurrent reviewers otherwise collide on the same
  files.
- >-
  The LAST task commits the fixes with the commit skill, as its own commit, separate from
  the implementation commit.
- >-
  Do NOT add an archive task. Do NOT add a task that pushes or opens a pull request -
  that stays a manual decision.
- >-
  openspec-reviewer-poc workflow v1 — end of managed block. Rules after this line, if any,
  are not managed by this install and must never be touched by an upgrade.
```
