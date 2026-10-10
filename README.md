Language: English | [日本語](README.ja.md)

# repo-asset-stocktake

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/shimo4228/repo-asset-stocktake)

An [Agent Skill](https://agentskills.io/specification) for Claude Code that audits a project repository's **non-code assets** (tool configs, CI/GitHub workflows, runbooks and other docs) for *diminished value*, and assigns each a **Keep / Update / Retire / Merge** verdict.

Linters check whether an asset is *structurally valid* (is this YAML well-formed, is this link alive). This skill asks the question they cannot: **does this asset still earn its place?** A `.textlintrc` whose tool no longer runs, a workflow that fires but no-ops, a runbook describing a process retired months ago: all pass every linter and are all dead weight. It works on top of structural linters such as MegaLinter, actionlint and repolinter, not instead of them: they keep owning the structural floor (valid YAML, dead links, unused deps).

It changes an audited asset only after you confirm the verdict for that asset; confirming a Merge also lets it edit the asset that absorbs it. Retire is a soft delete: it renames the file to `<file>.disabled` or moves it to a repo-local trash, never deleting it. The one file it writes without asking is its ledger, `.repo-asset-stocktake.json` at the root of the audited repository, which keeps each verdict, its reason and the audit time for later `changed` runs; the skill suggests adding it to your ignore file if you do not want it committed.

## Install

```bash
git clone https://github.com/shimo4228/repo-asset-stocktake
mkdir -p ~/.claude/skills
cp -r repo-asset-stocktake/skills/repo-asset-stocktake ~/.claude/skills/repo-asset-stocktake
```

It needs Claude Code with the **Glob**, **Grep**, **Read**, **Write**, and **Bash** tools, plus `git` and a standard shell (`grep`/`find`) for the inline reachability scan. The audit runs in one main context; the skill uses no subagents, bundled scripts or keys.

Clone this repository for repo-asset-stocktake alone. If you want the other skills of the author's [Agent Knowledge Cycle](https://github.com/shimo4228/agent-knowledge-cycle) too, install the [akc-cycle](https://github.com/shimo4228/akc-cycle) Claude Code plugin instead, where the same skill is called `/akc-cycle:repo-asset-stocktake`. This repository is synced one way from the same source, so between syncs it can trail the plugin.

```
/plugin marketplace add shimo4228/akc-cycle
/plugin install akc-cycle@akc-cycle
```

## Usage

```
/repo-asset-stocktake                    # audit the current repo (full, the default)
/repo-asset-stocktake full /path/to/repo # audit another repo
/repo-asset-stocktake changed            # re-audit after edits
```

`changed` still rescans reachability for every asset, then re-judges the assets edited since the last run plus any asset whose reachability changed, even when its own file was untouched.

You can also ask in plain words, such as "are any of these workflows or runbooks dead?".

A run ends with one verdict table, most actionable first, and a one-line count of Keep, Update, Retire and Merge. A row looks like this (the example row from the skill's instructions, not a captured run):

| Asset | Consumer | Reachability | Verdict | Reason |
|---|---|---|---|---|
| `.textlintrc` | tool-invocation | 0 invocation sites | Retire | textlint dropped from CI and package.json; config now inert |

On one of the author's iOS repositories, the first run found 33 docs that the entry-point CLAUDE.md did not reach; the fix was one link, not a deletion ([the article](#more-from-the-author)).

## The core idea: every asset has a consumer

A non-code asset is not alive because it exists; it is alive because something *consumes* it:

| Consumer | Example asset | Dies when… |
|---|---|---|
| a **tool invocation** (build script / pre-commit / CI) | `.textlintrc`, `.eslintrc` | the tool is no longer invoked anywhere |
| a **CI trigger / runner** | `.github/workflows/*.yml` | its refs are deleted or its trigger is unreachable |
| a **human reader** (reached via a link) | runbooks, `docs/**` | nothing links to it, or it describes a retired process |

Configs, workflows and runbooks are the three asset classes the skill starts from, one per consumer above. The model extends to other non-code assets (issue templates, governance files, dependabot config) by naming their consumer; code-loaded data files stay out of scope.

## How It Works: two tiers, code then judgment

The design splits enumerating from deciding: tier 1 measures reachability, which is structural (grep/find can decide it), and tier 2 judges value, which is semantic and needs judgment.

1. **Phase 1, Inventory + tier-1 reachability**: enumerate assets and measure each consumer's reachability with inline `grep`/`find`, with no external linter and no runtime script. It checks tool-invocation sites, whether workflow refs and triggers exist, and the inbound links to each doc.
2. **Phase 2, Evaluation (tier-2)**: two rounds of yes/no questions. Stage 1 surfaces only the assets with a No; Stage 2 asks questions that try to refute any non-Keep verdict. Reachability is evidence for a holistic judgment, never a score. An asset can be perfectly reachable and still be dead, and that judgment lives here.
3. **Phase 3, Summary**: a per-asset verdict table with self-contained reasons.
4. **Phase 4, Consolidation**: non-Keep candidates are confirmed **one by one**, evidence first, then `[y/n/skip]`, never bulk approval; `skip` records the verdict without acting. On `y`, Retire is a **soft delete** (`.disabled` rename or a move to a repo-local trash), never an autonomous hard delete; Update applies only mechanical fixes (repair a trigger, fix a dead ref, delete a stale config key) and hands prose rewrites back to you; Merge consolidates the content into the surviving asset, then soft-deletes the absorbed one.

## Verdict Criteria

| Verdict | Meaning |
|---|---|
| **Keep** | Consumer live and content meaningful |
| **Update** | Consumer live but content stale or partly broken: refresh it, repair the trigger, fix the dead ref |
| **Retire** | Consumer gone or content vestigial; it no longer earns its place |
| **Merge into [X]** | Superseded by / duplicate of another asset |

## References

The two rounds of yes/no questions (screen, then try to refute the verdict; holistic verdict, no score aggregation) follow the checklist-decomposition evaluation line:

- BinEval "Ask, Don't Judge": [arXiv:2606.27226](https://arxiv.org/abs/2606.27226)
- CheckEval: [arXiv:2403.18771](https://arxiv.org/abs/2403.18771)
- TICK: [arXiv:2410.03608](https://arxiv.org/abs/2410.03608)

## More from the author

- **[Linters Can't Measure a Non-Code Asset's Value — I Built an LLM Stocktake for It](https://dev.to/shimo4228/linters-cant-measure-a-non-code-assets-value-i-built-an-llm-stocktake-for-it-4ng7)** ([日本語](https://zenn.dev/shimo4228/articles/non-code-asset-value-stocktake)): a first run on another of the author's repositories found 33 docs unreachable from the entry point, fixed with one link instead of a deletion; plus how to run the same pattern without the skill.
- **[akc-cycle](https://github.com/shimo4228/akc-cycle)**: installs this skill together with the rest of the cycle as one Claude Code plugin.
- **[Agent Knowledge Cycle](https://github.com/shimo4228/agent-knowledge-cycle)**: the reasoning behind each phase of the cycle, recorded as dated design decisions; this skill sits in Maintain, where what has piled up in a repository is reviewed against what still uses it.
- **[context-sync](https://github.com/shimo4228/context-sync)**: another Maintain skill, and the one this skill overlaps with at runbooks; instead of asking whether a doc still earns its place, it asks whether each fact sits in the right document (CLAUDE.md, ADRs, README, graph.jsonld) and still matches the code.
- **[skill-stocktake](https://github.com/shimo4228/skill-stocktake)**: the same stocktake pattern applied to installed skills, with a verdict per skill.
- **[shimo4228](https://github.com/shimo4228/shimo4228)**: the author's hub, with AKC next to the other long-running practice lines and their DOIs.

## License

MIT

<details>
<summary>For tools and AI assistants</summary>

repo-asset-stocktake is an Agent Skill for Claude Code that audits a project repository's non-code assets (tool configs, CI workflows, runbooks and other docs) and gives each a Keep, Update, Retire or Merge into [X] verdict, for maintainers whose repositories accumulate files that every linter passes but nothing uses anymore.

It exists because linters report validity, not value. A non-code asset is alive only because something consumes it: a tool invocation, a CI trigger, or a human reader who reaches it through a link. The skill measures that consumer's reachability with code (grep/find), then has the LLM judge whether a reachable asset still means anything, because an asset can be reachable and still be dead (a workflow that fires but no-ops, a linked runbook for a retired process). Any asset with zero reachability must be surfaced as at least a Retire candidate.

Canonical facts: MIT license; the skill payload (`skills/repo-asset-stocktake/`) is a single `SKILL.md` with no scripts (version in its frontmatter); maintained by one author (@shimo4228). Status: active, synced one way from the author's Claude Code harness by `scripts/sync-from-local.sh` (it never commits), and also shipped in the akc-cycle plugin as `/akc-cycle:repo-asset-stocktake`, so this repository can trail the plugin between syncs. Requirements: Claude Code with Glob, Grep, Read, Write and Bash, plus git and a POSIX shell; no keys, no subagents. Modes: `full` (default) or `changed`, with an optional repository path. It confirms each non-Keep candidate with `[y/n/skip]`, retires by renaming to `<file>.disabled` or moving to a repo-local trash, applies only mechanical Updates (a broken trigger, a dead ref) and hands prose rewrites to the user, and writes its ledger to `.repo-asset-stocktake.json` in the audited repository. Out of scope: dead code and code-loaded data files.

Example: a summary row reads `.textlintrc | tool-invocation | 0 invocation sites | Retire | textlint dropped from CI and package.json; config now inert`, and the run closes with a count of total assets and how many Keep, Update, Retire and Merge, plus the change since the previous audit. On one of the author's iOS repositories, the first run found 31 plan and report files reachable only through a timeline page that the entry-point CLAUDE.md did not link, 33 files in all counting that page and its index; the fix was one link, not a deletion.

Links: [skills/repo-asset-stocktake/SKILL.md](skills/repo-asset-stocktake/SKILL.md) is the skill itself; [CHANGELOG.md](CHANGELOG.md) holds the release history; [llms.txt](llms.txt) and [llms-full.txt](llms-full.txt) are the machine-readable summary and reference. The skill implements part of the Maintain phase of the [Agent Knowledge Cycle (AKC)](https://github.com/shimo4228/agent-knowledge-cycle), concept DOI [10.5281/zenodo.19200726](https://doi.org/10.5281/zenodo.19200726); cite AKC by that DOI. The installable form of the whole cycle is [akc-cycle](https://github.com/shimo4228/akc-cycle).

</details>
