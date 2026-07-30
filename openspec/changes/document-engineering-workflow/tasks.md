## 1. Branch

- [x] 1.1 Resolve the branch per `rules.tasks` in `openspec/config.yaml` before writing anything. Current branch was `docs/add-readme`, not a work-branch prefix, so a branch was created off it.

## 2. Implementation

- [x] 2.1 Rewrite `README.md` "how it works": where the workflow is declared, how `rules.tasks` reaches `tasks.md`, and the steps in execution order
- [x] 2.2 Add the review section: how each of the three reviews is invoked and what scope each one reads
- [x] 2.3 Add the decision record: why the commit precedes the reviews, why `/code-review` runs as a subprocess
- [x] 2.4 Add the rejected-options section, each stating what it was, why it was dropped, what was lost, and how to rebuild it

## 3. Verification

- [x] 3.1 Exercise the change with whatever this environment offers. No app to run, so the executable claims were checked directly: the Codex glob resolves, both shell snippets pass `bash -n`, the internal link resolves, and spec keywords are present.

## 4. Commit implementation

- [x] 4.1 Commit the README with the commit skill, before the reviews run

## 5. Review

- [x] 5.1 Run /security-review via the Skill tool. Resolve `origin/HEAD` first; with no remote at all, skip and report the reduced count instead of a pass.
- [x] 5.2 Run `claude -p '/code-review high'` via Bash
- [x] 5.3 Resolve `C=$(ls -d ~/.claude/plugins/cache/openai-codex/codex/*/scripts/codex-companion.mjs 2>/dev/null | sort -V | tail -1)`; empty means skip. Otherwise `node "$C" adversarial-review --scope branch --base main` — branch scope, since the working tree is clean after step 4.

## 6. Fixes

- [x] 6.1 Apply fixes for findings from all three reviews, after every review has finished

## 7. Commit fixes

- [x] 7.1 Commit the fixes with the commit skill, as a separate commit
