# openspec-reviewer-poc

The engineering workflow for this repo — branch, implement, verify, commit, review, fix,
commit — is declared in [`openspec/config.yaml`](openspec/config.yaml) instead of being
pasted into the agent as a prompt on every task.

This document covers **how that works** first, then the **decision record** behind it: what
was considered, what was rejected, and why.

This is a proof-of-concept repository: it holds no application code and exists to host and
exercise the workflow itself.

## Installation via agent

Paste this into any Claude Code session in the project you want to adopt the workflow:

```
Install this workflow into my project. Download these files with curl — not WebFetch, which
summarises instead of returning the file verbatim:

https://raw.githubusercontent.com/DrSmile444/openspec-reviewer-poc/refs/heads/main/openspec/config.yaml
https://raw.githubusercontent.com/DrSmile444/openspec-reviewer-poc/refs/heads/main/openspec/workflow-rules.md
https://raw.githubusercontent.com/DrSmile444/openspec-reviewer-poc/refs/heads/main/.claude/skills/create-branch/SKILL.md
https://raw.githubusercontent.com/DrSmile444/openspec-reviewer-poc/refs/heads/main/.claude/skills/commit/SKILL.md

If openspec/config.yaml does NOT exist yet: write the downloaded config.yaml and
workflow-rules.md verbatim to the same paths here.

If openspec/config.yaml already EXISTS: do not overwrite it. Instead:
1. Read my existing rules.tasks list and look for a line containing the substring
   "openspec-reviewer-poc workflow v".
2. Same version number as the marker in the downloaded workflow-rules.md → make no changes,
   tell me it's already installed, stop.
3. Older version number → show me the existing block and the new one from
   workflow-rules.md, and ask before replacing the old one.
4. No marker found → append the entire rules.tasks block from workflow-rules.md to the end
   of my existing rules.tasks list, and append its rules.proposal bullet to my existing
   rules.proposal list only if no equivalent rule is already there. Do not touch schema,
   context, external_docs, or any rules key other than tasks/proposal. Show me a diff of
   ONLY the lines you're adding — not a full-file diff — and ask before writing.

Either way, write the two skills as a plain overwrite — they're self-contained utility
skills, not project data.
```

`config.yaml` carries the workflow itself; `workflow-rules.md` is the portable payload used
for merging into an existing `config.yaml`; the two skills are what the branch and commit
steps call by name. Without them those steps still run, but the agent improvises them.

Same thing without an agent — **fresh installs only** (no existing `openspec/config.yaml`;
merging into an existing one needs the judgment calls above, so it stays an agent-only
operation):

```bash
B=https://raw.githubusercontent.com/DrSmile444/openspec-reviewer-poc/refs/heads/main
mkdir -p openspec .claude/skills/create-branch .claude/skills/commit
curl -sL "$B/openspec/config.yaml"                  -o openspec/config.yaml
curl -sL "$B/openspec/workflow-rules.md"             -o openspec/workflow-rules.md
curl -sL "$B/.claude/skills/create-branch/SKILL.md" -o .claude/skills/create-branch/SKILL.md
curl -sL "$B/.claude/skills/commit/SKILL.md"        -o .claude/skills/commit/SKILL.md
```

Two notes:

- **Install the skills globally instead** by pointing the last two at `~/.claude/skills/`, if
  you want the workflow in every project rather than this one. If the new skills do not show
  up straight away, `/reload-skills`.
- **The rules only take effect on newly generated changes.** They are injected at propose
  time, so an existing `tasks.md` keeps whatever it was built with.

Afterwards, `/validate-openspec-config` (if you have it) confirms every rule actually reaches
the CLI rather than being silently dropped.

### Re-running the install

The `rules.tasks` block carries a version marker (`openspec-reviewer-poc workflow vN`), so
re-pasting the same agent prompt on a project that already has it installed is a no-op — it
won't duplicate the branch/review instructions. See
[`openspec/workflow-rules.md`](openspec/workflow-rules.md) for how the marker and the merge
logic work.

## Layout

- `openspec/config.yaml` — schema, project context, and the per-artifact rules that carry the
  workflow
- `openspec/workflow-rules.md` — the portable `rules.tasks`/`rules.proposal` payload, kept in
  sync with `config.yaml`, and what an install merges into an existing project's config
- `openspec/changes/` — active changes
- `openspec/specs/` — accepted specifications
- `.claude/skills/create-branch`, `.claude/skills/commit` — the two skills the workflow calls
  by name, vendored so the repo is self-contained
- `.claude/skills/openspec-*`, `.claude/commands/opsx/` — the OpenSpec commands themselves
- `ENGINEERING_PROMPT.md` — the hand-pasted prompt this workflow replaces, kept for reference

---

# Part 1 — How it works

## The mechanism

OpenSpec's project config accepts exactly three fields: `schema`, `context`, and `rules`
(per-artifact). `schema` selects the workflow schema and is read by `openspec new change`;
`context` and `rules` are consumed by `openspec instructions <artifact> --json`, which means
they influence **artifact generation only**. There are no hooks around apply, no git
integration, no orchestration.

So the workflow is not executed by the config. It is *compiled into* each change's
`tasks.md` at propose time, and the apply phase then walks that checklist.

```
openspec/config.yaml
  rules.proposal  ──┐
  rules.tasks     ──┤
                    │  injected by `openspec instructions <artifact> --json`
                    ▼
            /opsx:propose  →  proposal.md, design.md, specs/, tasks.md
                                                          │
                                                          │  parsed as checkboxes
                                                          ▼
                                                    /opsx:apply
                                                          │
                                            invokes the real skills and commands
```

Two consequences worth knowing:

- **Rules are compile-time.** Editing `config.yaml` does not change a `tasks.md` that already
  exists. Regenerate the change to pick up new rules.
- **Rules are guidance, not enforcement.** The model generating `tasks.md` is asked to follow
  them; nothing validates that it did. Read the generated `tasks.md` before applying.

## The steps

| # | Step | Runs via |
| --- | --- | --- |
| 1 | Branch (conditional) | `create-branch` skill |
| 2 | Implementation | normal editing |
| 3 | Verification | `run` skill / probe scripts |
| 4 | **Commit the implementation** | `commit` skill |
| 5 | Three reviews, in parallel | see below |
| 6 | Fixes, after all three finish | normal editing |
| 7 | Commit the fixes, separately | `commit` skill |

There is no archive step and no pull-request step. Both were deliberate removals — see the
decision record.

## Step 1 — the branch decision

`rules.tasks` in [`openspec/config.yaml`](openspec/config.yaml) is the source of truth; the
diagram below is a reading aid. If the two disagree, the config wins and this section is
stale.

The goal is to avoid two failure modes: committing onto a shared branch, and spawning a
redundant branch when the current one is already the right place.

```
current branch matches <feature|bugfix|hotfix|chore>/... ?
│
├─ NO ─────────────────────────────────────────────► create a branch
│      (main, master, develop, uat, staging,
│       release/*, or any unprefixed name)
│
└─ YES
    ├─ 0 commits ahead of trunk AND no open PR ────► reuse
    ├─ branch task ID == `Task:` in proposal.md ───► reuse
    └─ otherwise ──────────────────────────────────► show facts, ask the user
                                                     (branch name, commits ahead,
                                                      open PR, new change name)
```

Two details that matter in practice:

- **New branches default to the current branch as their base, not trunk.** Stacked branches
  are the expected shape here: doing several tasks back to back means later PRs contain
  earlier commits, which is accepted. On the *ask* path the user picks the base, and their
  choice wins over this default.
- **`git pull --ff-only` is skipped when the chosen base has no upstream.** The
  `create-branch` skill deliberately never pushes, so a stacked branch has no tracking
  branch, and the skill's default path would stop on every second task.

The task tracker ID is written into `proposal.md` as a `Task:` line at propose time. It has
to live in the file rather than in conversation context, because apply may run in a fresh
session where the conversation is gone.

## Step 5 — the three reviews

Each is invoked differently, and — this is the part that is easy to get wrong — **each reads
a different scope**.

The three are independent and are launched together, not one after another. The checklist in
`tasks.md` is flat only because that is how it represents tasks.

| Review | Invoked by | Reads |
| --- | --- | --- |
| `/security-review` | `Skill` tool | **committed history only** (`git diff origin/HEAD...`) |
| `/code-review high` | `claude -p '/code-review high'` via Bash | branch commits **+ working tree** |
| Codex adversarial | `codex-companion.mjs adversarial-review --scope branch` | branch commits vs trunk |

Neither `/code-review` nor `/codex:adversarial-review` can be called through the `Skill`
tool — both are marked `disable-model-invocation`. The documented escape hatch differs per
tool: a headless `claude -p` subprocess for the former, the plugin's own script for the
latter.

**Codex uses `--scope branch`, not `--scope working-tree`.** The implementation is committed
in step 4, so by review time the working tree is clean and a working-tree review would
inspect nothing. The script is resolved without hardcoding a version:

```bash
C=$(ls -d ~/.claude/plugins/cache/openai-codex/codex/*/scripts/codex-companion.mjs \
      2>/dev/null | sort -V | tail -1)
[ -n "$C" ] && node "$C" adversarial-review --scope branch --base "$TRUNK"
```

An **empty** result means the plugin is absent: skip and report the reduced review count. A
**non-empty** result followed by a non-zero exit means the review *failed* — not the same
thing, and reported as a failure rather than filed under "plugin missing". Neither case
blocks the workflow.

**`/security-review` needs `origin/HEAD` to resolve.** It builds its diff against
`origin/HEAD...`, which hard-fails with `fatal: ambiguous argument` when the ref is missing.
The workflow tries `git remote set-head origin -a` first; if the repo has no remote at all,
the review is skipped and the reduced count reported — never reported as a pass.

## Dependencies

Nothing is installed and nothing is generated outside this repo. The workflow references only
what already exists on the machine:

- `create-branch`, `commit` — vendored into [`.claude/skills/`](.claude/skills/), so a clone
  has them without any personal setup
- `run` — built-in skill
- `/security-review`, `/code-review` — built into the Claude Code binary
- `codex` plugin — optional; the flow degrades to two reviews without it

A project skill shadows a same-named one in `~/.claude/skills/`, so inside this repo the
vendored copies win. That is the point — the repo is self-contained — but it also means a
later change to your personal copy will not reach here. Re-copy when you change them.

---

# Part 2 — Decision record

Every claim below was verified against a live repo with planted defects, on **2026-07-30**,
with Claude Code `2.1.220`, OpenSpec CLI `1.4.1`, and the `codex` plugin `1.0.4`. Tool
behaviour can change with a release; re-verify before trusting any of it years from now.

## Why the implementation commit comes before the reviews

This is the least obvious choice in the config, and it inverts the original hand-written
prompt, which reviewed before committing.

The built-in `/security-review` builds its diff from these commands, extracted from the
Claude Code binary:

```
!`git status`
!`git diff --name-only origin/HEAD...`
!`git diff origin/HEAD...`              ← three dots: commit-to-commit
!`git log --no-decorate origin/HEAD...`
```

Three-dot syntax compares commits. The working tree is invisible to it. Tested directly: with
an untracked file containing a command injection and a hardcoded key, plus an uncommitted
edit adding a token to a tracked file, **neither appeared in the diff the review received** —
while its prompt still asserted "This contains all code changes in the PR."

`GIT STATUS` does list the filenames, so a reviewer might notice something is missing, but
the content never reaches it.

So reviewing before committing meant `/security-review` reviewed an empty diff and reported
success. Moving the commit ahead of the reviews fixes it with no cleverness.

**Is committing first dangerous?** The worry is a secret landing in git history. In practice
it does not, because this workflow never pushes — the PR step was removed on purpose. A
secret in a local, unpushed commit is removed with `git reset`. It has not left the machine.

## Why `/code-review` runs as a subprocess

`/code-review` cannot be invoked by the model. Attempting it returns:

```
Skill code-review cannot be used with Skill tool due to disable-model-invocation
```

This is intentional, and documented:

> You can't schedule the review: `/code-review` is marked `disable-model-invocation`, so if
> you set it as a scheduled task's prompt, Claude reads it as plain text instead of running
> the review.
> — [Code Review docs](https://code.claude.com/docs/en/code-review)

What is blocked is the *tool*, not the command. The same docs describe running it headlessly
(`claude -p '/code-review ultra'` for CI), and that path works: `claude -p '/code-review
high'` returned findings in `ReportFindings` JSON, including a hardcoded key in an
**untracked** file and a stray line in an **uncommitted** edit.

That also makes `/code-review` the broadest of the three reviews — it is the only one that
natively reads the working tree and reports hardcoded secrets.

## Why `/security-review` does not catch secrets

Worth knowing so it is not mistaken for a bug. Its prompt carries a hard exclusion:

```
2. Secrets or credentials stored on disk (these are handled by other processes)
```

Verified: on a snapshot containing a GitHub token, a database password, and a command
injection, it reported **only the command injection**. Both secrets were filtered out by
design — Anthropic delegates secrets to dedicated scanners.

The gap is covered by `/code-review`, which has no such exclusion.

## Why there is no archive step

Removed at the maintainer's request. Archiving is available as `/opsx:archive` when wanted.

## Why there is no pull-request step

A PR belongs to a *branch*; an OpenSpec change belongs to a *task*. Running several changes
on one branch would mean a PR per change, which is wrong.

The stacked-branch decision weakens that argument — one change per branch makes them line up
again — but introduces a timing problem instead: a PR opened on a stacked branch contains the
commits of every branch below it until those merge. Picking the right moment is a human call,
so the workflow stops after the fix commit and leaves `pr-description` / `share-pr` to be
invoked manually. That is also where the push happens, since `create-branch` never pushes.

---

# Part 3 — Rejected options

Recorded with enough detail to rebuild, should the trade-off change.

## A forked `security-review` command

**What it was.** `.claude/commands/sec-review-wt.md`, derived from
[`anthropics/claude-code-security-review`](https://github.com/anthropics/claude-code-security-review),
with three deltas:

1. **Scope.** Replace `git diff origin/HEAD...` with a base-resolving pair that reads the
   working tree:
   ```bash
   B=$(git rev-parse --verify -q origin/HEAD >/dev/null 2>&1 && echo origin/HEAD \
       || (git rev-parse --verify -q main >/dev/null 2>&1 && echo main || echo master))
   git diff --merge-base "$B"                                   # tracked, incl. uncommitted
   for f in $(git ls-files --others --exclude-standard); do     # untracked
     git diff --no-index -- /dev/null "$f"
   done
   ```
   `--no-index` leaves the index untouched. (`git add -N .` also exposes untracked files to
   `git diff`, but leaves intent-to-add entries that need `git reset` afterwards.) Resolving
   the base through a fallback chain also fixes a real failure: upstream hard-fails with
   `fatal: ambiguous argument 'origin/HEAD...'` in a repo with no remote.

2. **Secrets back in scope.** Delete the `secrets stored on disk` exclusion. Upstream assumes
   a separate scanner handles them; a pre-commit review is exactly where a hardcoded
   credential should be caught.

3. **A `Committed: yes|no` field** per finding, plus a closing warning when any finding is
   `no` — so it is obvious what can still simply be deleted.

**It worked.** Registered as a model-invocable skill immediately, without a session restart,
and caught all four planted defects — including `DB_PASS = "SuperSecret123!"`, which nothing
else caught.

**Why it was dropped.** It is a fork of a prompt maintained by someone else. The built-in
commands update with Claude Code; a fork updates only when someone remembers it exists. In
six months it becomes a dead file that still *looks* like protection. Three live tools with a
known gap beat four where one has quietly rotted.

**What was lost.** Free-form credentials with no recognisable format. `/code-review` catches
structured secrets (`sk-live-…`, `ghp_…`); whether it catches an arbitrary password like
`SuperSecret123!` was never tested. Neither the built-in `/security-review` nor a scanner
will.

## The snapshot trick

**What it was.** Keep the built-in `/security-review` unmodified and change what it looks at:

```bash
git add -A && git commit -q -m "tmp: pre-review snapshot"
claude -p '/security-review'
git reset -q --soft HEAD~1; git reset -q
```

**It worked.** After the snapshot the built-in command sees untracked and uncommitted work;
the reset restores the branch and working tree exactly.

**Why it was not adopted.** It was pitched as a way to keep secrets out of git — but the
snapshot itself writes a commit object containing the secret. The supposed advantage over
simply committing first is nil, and it adds a git dance that leaves a stray `tmp:` commit if
anything fails midway.

**What was lost.** The ability to review before any commit object exists. Irrelevant while
nothing is pushed automatically, but it is the right answer the moment that changes — which
is why the recipe is kept above rather than deleted.

## A dedicated secret scanner

**What it was.** `npx secretlint` as a deterministic gate, needing no install:

```bash
echo '{"rules":[{"id":"@secretlint/secretlint-rule-preset-recommend"}]}' > .secretlintrc.json
npx --yes -p secretlint -p @secretlint/secretlint-rule-preset-recommend secretlint "**/*"
```

**It worked, partially.** Caught `ghp_…` in an uncommitted README edit without a commit or an
install. Did **not** catch `DB_PASS = "SuperSecret123!"` — scanners match formats, not
meaning.

**Why it was not adopted.** It mostly duplicates what `/code-review` already reports, and adds
a network dependency to every run.

**What was lost.** Determinism. Every remaining check is an LLM judgement that can vary run to
run; a scanner either matches the pattern or does not. Worth revisiting if this repo ever
gains CI, where that property matters more than it does locally.

## A wrapper skill around the reviews

**What it was.** A project skill owning review orchestration — Codex detection, parallel
fan-out, degradation to two reviews.

**Why it was dropped.** Once `claude -p` and the Codex script turned out to be plain Bash
calls, there was nothing left for a wrapper to encapsulate. Three lines in `rules.tasks` do
the same job with one less file to maintain.

**What was lost.** A single place to change review behaviour. The invocations now live in
`rules.tasks`, so a change to how reviews run means editing prose in the config rather than
one file with a name.

## Enumerating trunk branch names

**What it was.** Deciding "create a branch" by matching against `main`, `master`, `uat`, and
so on.

**Why it was dropped.** The list never ends — `develop`, `staging`, `qa`, `preprod`,
`release/2.1`, someone's `dmytro-test`. Inverting it is closed-ended: a branch is a *work*
branch when it carries a `feature|bugfix|hotfix|chore` prefix, and everything else is treated
as shared. `release/*` has a prefix but is deliberately excluded.

**What was lost.** Work branches using a prefix outside that set — `docs/…`, `spike/…` — are
now treated as shared and trigger a new branch. Widening the set is a one-word edit in
`rules.tasks` if that becomes annoying.

## Semantic branch matching

**What it was.** Comparing the current branch's description against the new change's
description to decide reuse.

**Why it was demoted.** It is the least reliable and least auditable signal available. Three
objective ones come first — commits ahead of trunk, an open PR, a matching task ID — and they
settle most cases. When they do not, the workflow shows the facts and asks, because a wrong
guess is expensive in both directions: a polluted PR, or orphaned work.

**What was lost.** Full automation of the ambiguous case. Continuing related work on an
existing untracked-ticket branch now costs one question instead of being inferred.

## Treating every Codex failure as "plugin missing"

**What it was.** The first draft collapsed any non-zero exit into a skip.

**Why it was fixed.** Codex itself flagged it during an adversarial review of this very
config:

> That collapses distinct failure modes into a skip: a missing plugin, a broken invocation
> path, a crashed review, or a review command that signals failure can all be ignored […] a
> real review failure can be bypassed.

Now the file check and the execution are separate: absence is a skip, a non-zero exit is a
reported failure. Neither blocks, but they are no longer indistinguishable.

**What was lost.** Nothing. The earlier form was simpler to write but reported a review that
had not happened as though it had, which is worse than the extra line it saved.
