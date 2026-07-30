## 1. Branch

- [ ] 1.1 Resolve the branch before writing anything. Current branch is a work branch only if it matches `<feature|bugfix|hotfix|chore>/...`; otherwise create one with the create-branch skill, based on the current branch, slug `document-engineering-workflow`. On a work branch, reuse it if it is 0 commits ahead of trunk with no open PR or its task ID matches `Task:` in proposal.md; otherwise show the facts and ask. Skip `git pull --ff-only` if there is no upstream.

## 2. Implementation

- [ ] 2.1 Rewrite `README.md` "how it works": where the workflow is declared, how `rules.tasks` reaches `tasks.md`, and the steps in execution order
- [ ] 2.2 Add the review section: how each of the three reviews is invoked and whether it reads committed history or the working tree
- [ ] 2.3 Add the decision record: why the commit precedes the reviews, why `/code-review` runs as a subprocess
- [ ] 2.4 Add the rejected-options section, each with what it was, why it was dropped, what was lost, and how to rebuild it

## 3. Verification

- [ ] 3.1 Exercise the change with whatever this environment offers. No app to run here, so verify by re-reading `README.md` against the spec scenarios and confirming every claim about tool behaviour matches what was actually observed. Say so explicitly if something cannot be checked.

## 4. Commit implementation

- [ ] 4.1 Commit the README with the commit skill, before the reviews run

## 5. Review

- [ ] 5.1 Run /security-review via the Skill tool
- [ ] 5.2 Run `claude -p '/code-review high'` via Bash
- [ ] 5.3 Resolve `C=$(ls -d ~/.claude/plugins/cache/openai-codex/codex/*/scripts/codex-companion.mjs 2>/dev/null | sort -V | tail -1)`. Empty means plugin absent: skip, note "2 of 3", continue. Otherwise run `node "$C" adversarial-review --scope working-tree`; a non-zero exit is a review failure, not an absent plugin — report it, do not block.

## 6. Fixes

- [ ] 6.1 Apply fixes for findings from all three reviews, after every review has finished

## 7. Commit fixes

- [ ] 7.1 Commit the fixes with the commit skill, as a separate commit
