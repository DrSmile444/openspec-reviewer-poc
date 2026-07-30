## 1. Branch

- [ ] 1.1 If currently on `main` or `master`, create a branch using the create-branch skill. If already on a feature branch for this work, reuse it and skip.

## 2. Implementation

- [ ] 2.1 Create `README.md` at the repo root stating the repository's purpose in the first paragraph
- [ ] 2.2 Add a section to `README.md` pointing to `openspec/config.yaml` as the place where the branch → implement → review → commit workflow is declared

## 3. Review

- [ ] 3.1 Run /security-review
- [ ] 3.2 Run /code-review high
- [ ] 3.3 Run codex adversarial review: `node "$(ls -d ~/.claude/plugins/cache/openai-codex/codex/*/ | sort -V | tail -1)scripts/codex-companion.mjs" adversarial-review --scope working-tree`. If the command exits non-zero, treat it as "codex plugin not installed" and continue.

## 4. Fixes

- [ ] 4.1 Apply fixes for all findings from tasks 3.1–3.3

## 5. Commit

- [ ] 5.1 Commit using the commit skill
