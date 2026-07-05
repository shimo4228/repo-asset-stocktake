# Changelog

All notable changes to this skill are documented here. Format loosely follows
[Keep a Changelog](https://keepachangelog.com/); the skill is versioned in its
`SKILL.md` frontmatter (`metadata.version`).

## [1.0] — initial release

First public release of `repo-asset-stocktake`.

- Audits a project repo's **non-code assets** (tool configs, CI/GitHub
  workflows, runbooks/docs) for *diminished value*, assigning
  **Keep / Update / Retire / Merge** verdicts.
- **Two-tier design**: tier-1 enumerates consumer-reachability with inline
  `grep`/`find` (structural, no external linter, no runtime script); tier-2
  applies holistic value judgment to the survivors (semantic, no score).
- **Consumer-class model**: every non-code asset serves a consumer (tool
  invocation / CI trigger / human navigation); configs/workflows/runbooks are
  extensible seed classes, not a closed set.
- **Confirm-each Phase 4** with soft-delete-first Retire (never an autonomous
  hard-delete).
- Sibling of `skill-stocktake` and `rules-stocktake`; implements the
  **Maintain** phase of the Agent Knowledge Cycle.
