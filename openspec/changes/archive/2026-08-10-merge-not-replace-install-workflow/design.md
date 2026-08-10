## Context

See `proposal.md` - Why. `openspec/config.yaml` currently bundles two things that install
must treat differently: project identity (`schema`, `context`, non-`tasks`/`proposal` rules)
and the portable workflow payload (`rules.tasks`, one `rules.proposal` bullet). The install
prompt in `README.md` currently treats the whole file as one unit to copy.

## Goals / Non-Goals

**Goals:**
- Make re-installing into an existing `config.yaml` additive and safe for arbitrary schemas
  (verified against a real project using a custom `flowforge` schema with its own `context`,
  `design`/`specs`/`adrs`/`plan`/`self_review` rules).
- Make re-running the install prompt idempotent: same result whether run once or twice.
- Keep the fresh-install path (no existing `config.yaml`) exactly as simple as it is today.

**Non-Goals:**
- Per-bullet semantic merging (comparing each of the workflow's rule sentences against the
  target's existing rules for equivalent meaning). Rejected as unreliable — an LLM judging
  "this existing rule already covers it" can misjudge in both directions. The payload is
  installed as one atomic, versioned block instead.
- A YAML-aware non-agent merge tool (e.g. `yq`). Rejected because it adds an external
  dependency to a repo whose stated design is that nothing is installed outside it
  (`README.md` - Dependencies). Merging into an existing file stays an agent-only operation.
- Automatic migration of a target's `tasks.md` that was already generated under an older
  workflow version. Per `README.md` - "Rules are compile-time", regenerating a change is the
  existing, documented way to pick up new rules; this change doesn't alter that.

## Decisions

**Extract the payload into `openspec/workflow-rules.md`, not inline in the README prompt.**
Keeping it as a standalone file means `config.yaml`'s own `rules.tasks` and the README's
install prompt both point at the same source instead of two copies drifting apart. The file
holds the exact `rules.tasks` array content (as a YAML list) and the `rules.proposal` bullet,
plus the version marker.

**Version marker is a plain sentence prefix, not YAML metadata.** `rules.tasks` is a flat
list of prose strings — there is nowhere else in that structure to carry version info. The
marker becomes the literal first bullet: `openspec-reviewer-poc workflow v1 — branch,
implement, verify, commit, review, fix, commit.` Detection is a substring check for
`openspec-reviewer-poc workflow v` against the target's `rules.tasks` array, then comparing
the trailing version number. Alternative considered: a YAML comment (`# workflow-version: 1`)
above the list — rejected because `openspec instructions --json` may not round-trip comments,
and the marker needs to survive being read back by a future install.

**Whole-block atomic replace/append, never per-line diffing against the target's existing
rules.** When no marker is found, the entire payload is appended as-is. When an older marker
is found, the entire old payload block is identified (from its marker line to the end of the
contiguous run of lines this workflow owns) and replaced as a unit. This avoids the tool
having to reconcile partial hand-edits a user may have made to individual bullets.

**The "process scaffolding, not plan-derived tasks" clarification lives in the payload
itself, not as schema-specific logic in the install prompt.** The install prompt has no way
to know every schema's task-derivation rules in advance. One sentence in the payload, true
regardless of target schema, is cheaper than detecting and special-casing schemas that
require task-to-plan traceability.

**Diff shown to the user is computed from the merge result, not the raw downloaded file.**
The install prompt must diff "target's `rules.tasks`/`rules.proposal` before" against
"...after append/replace" — scoped to those two keys — rather than diffing the whole
downloaded `config.yaml` against the whole target file, which is what produced the
full-replacement diff this change fixes.

## Risks / Trade-offs

[Risk] The version-marker substring could theoretically collide with unrelated user text →
Mitigation: the marker string (`openspec-reviewer-poc workflow v`) includes the repo name,
making an accidental collision implausible; not worth defending against further.

[Risk] A user who hand-edited the installed block (e.g. removed one review step) will have
their edit silently blown away on the next version-bump replace → Mitigation: the install
prompt shows the old-vs-new diff and asks before replacing an older version; it does not
auto-replace silently.

[Risk] `rules.tasks` structure could differ across schemas in ways not seen yet (e.g. a
schema without a `tasks` artifact at all, or a nested structure instead of a flat list) →
Mitigation: out of scope for this change; the install prompt should check that `rules.tasks`
resolves to a flat list of strings before attempting to append, and fall back to asking the
user if it does not.

## Open Questions

- Whether to bump the version marker on every future wording tweak to the payload, or only
  on changes that affect installed behavior. Doesn't change this change's specs or tasks —
  can be decided the next time the payload actually changes.
