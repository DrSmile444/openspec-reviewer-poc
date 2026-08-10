## 1. Branch

- [x] 1.1 Resolve a work branch for this change per the branch decision rule (create one with
  the create-branch skill if the current branch is not a work branch; otherwise reuse or ask
  as the rule specifies).

## 2. Extract the portable workflow payload

- [x] 2.1 Create `openspec/workflow-rules.md` holding the `rules.tasks` array content (as a
  YAML-list-ready block) and the `rules.proposal` bullet, with the version marker
  `openspec-reviewer-poc workflow v1 — branch, implement, verify, commit, review, fix,
  commit.` as the first line of the `rules.tasks` payload.
- [x] 2.2 Add one sentence to the payload text stating that its branch/verify/commit/review/
  fix/commit steps are process scaffolding around the target's own implementation tasks, not
  tasks derived from a plan or spec themselves.
- [x] 2.3 Update `openspec/config.yaml`'s `rules.tasks` to include the same version-marker
  first line, keeping the rest of its content unchanged, so this repo's own config and
  `workflow-rules.md` stay in sync.

## 3. Rewrite the install prompt in README.md

- [x] 3.1 Rewrite the "Installation via agent" paste-in prompt so that when
  `openspec/config.yaml` already exists in the target project, it instructs the agent to:
  read the file, check `rules.tasks` for the version marker, and branch three ways (same
  version → no-op/report; older version → show old-vs-new block and ask before replacing; no
  marker → append the payload to `rules.tasks` and the Task-ID bullet to `rules.proposal` if
  missing) while leaving `schema`, `context`, and every other rules key untouched.
- [x] 3.2 Update the prompt so the diff shown to the user before writing is scoped to only the
  added/changed lines in `rules.tasks`/`rules.proposal`, not a full-file diff.
- [x] 3.3 Narrow the "Same thing without an agent" bash-only section to the fresh-install case
  (no existing `config.yaml`) and add a line stating that merging into an existing file
  requires the agent-based path.

## 4. Verification

- [x] 4.1 Verify `openspec/config.yaml` and `openspec/workflow-rules.md` both parse as valid
  YAML/Markdown and their `rules.tasks` payloads match, by reading both files back.
- [x] 4.2 Dry-run the rewritten install prompt's merge logic by hand against the sample
  config shape described in `specs/workflow-installation/spec.md` (a `config.yaml` with a
  different `schema`, a `context` block, and `design`/`specs`/`adrs` rules) and confirm the
  scenarios in that spec hold: unrelated keys stay untouched, and a second run is a no-op.
- [x] 4.3 Run `openspec validate --strict` for this change and fix anything it flags.

## 5. Commit the implementation

- [x] 5.1 Commit the implementation with the commit skill.

## 6. Review

- [x] 6.1 Review (a): run `/security-review` through the Skill tool (resolve `origin/HEAD`
  first; skip and report a reduced count if there is no remote).
- [x] 6.2 Review (b): run `claude -p '/code-review high'` through Bash.
- [x] 6.3 Review (c): run the Codex adversarial review via `codex-companion.mjs
  adversarial-review --scope branch --base <trunk>` if the plugin script resolves; skip and
  note it in the review count if it does not.

## 7. Fix

- [x] 7.1 Apply fixes for the findings from all three reviews, once every review has
  finished.

## 8. Commit the fixes

- [x] 8.1 Commit the fixes with the commit skill, as a separate commit from the
  implementation commit.
