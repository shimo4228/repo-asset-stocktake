Language: English | [日本語](README.ja.md)

# repo-asset-stocktake

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/shimo4228/repo-asset-stocktake)

An [Agent Skill](https://agentskills.io/specification) that audits a project repository's **non-code assets** — tool configs, CI/GitHub workflows, runbooks and other docs — for *diminished value*, and assigns each a **Keep / Update / Retire / Merge** verdict.

Linters check whether an asset is *structurally valid* (is this YAML well-formed, is this link alive). This skill asks the question they cannot: **does this asset still earn its place?** A `.textlintrc` whose tool no longer runs, a workflow that fires but no-ops, a runbook describing a process retired months ago — all pass every linter and are all dead weight.

## The core idea — every asset has a consumer

A non-code asset is not alive because it exists; it is alive because something *consumes* it:

| Consumer | Example asset | Dies when… |
|---|---|---|
| a **tool invocation** (build script / pre-commit / CI) | `.textlintrc`, `.eslintrc` | the tool is no longer invoked anywhere |
| a **CI trigger / runner** | `.github/workflows/*.yml` | its refs are deleted or its trigger is unreachable |
| a **human reader** (reached via a link) | runbooks, `docs/**` | nothing links to it and it describes a retired process |

configs / workflows / runbooks are three **seed** consumer-classes — the model extends to any asset (issue templates, governance files, dependabot config) by naming its consumer.

## Install

### Claude Code

```bash
cp -r skills/repo-asset-stocktake ~/.claude/skills/repo-asset-stocktake
```

### SkillsMP

```bash
/skills add shimo4228/repo-asset-stocktake
```

## Usage

```
/repo-asset-stocktake                    # audit the current repo
/repo-asset-stocktake full /path/to/repo # audit another repo
/repo-asset-stocktake changed            # re-audit only what changed since last run
```

## How It Works — two tiers, code then judgment

The design follows the **enumerate / decide** split: reachability is structural (decidable by grep/find), value is semantic (needs judgment).

1. **Phase 1 — Inventory + tier-1 reachability**: enumerate assets and measure each consumer's reachability with inline `grep`/`find` — no external linter, no runtime script. Tool-invocation sites, workflow ref/trigger existence, doc inbound-link graphs.
2. **Phase 2 — Evaluation (tier-2)**: a two-stage binary screen (Stage 1 Yes/No, surface only the No; Stage 2 refutation questions pressure-test any non-Keep verdict). Reachability is evidence for a holistic judgment, never a score. An asset can be perfectly reachable and still be dead — that judgment lives here.
3. **Phase 3 — Summary**: a per-asset verdict table with self-contained reasons.
4. **Phase 4 — Consolidation**: non-Keep candidates confirmed **one by one** — evidence first, then `[y/n/skip]`, never bulk approval. Retire is a **soft-delete first** (`.disabled` rename), never an autonomous hard-delete.

## Verdict Criteria

| Verdict | Meaning |
|---|---|
| **Keep** | Consumer live and content meaningful |
| **Update** | Consumer live but content stale or partly broken — refresh it, repair the trigger, fix the dead ref |
| **Retire** | Consumer gone or content vestigial — no longer earns its place |
| **Merge into [X]** | Superseded by / duplicate of another asset |

## Why not a structural linter (MegaLinter, actionlint, repolinter)?

Those are excellent and this skill does not replace them — they own the **structural** floor (valid YAML, dead links, unused deps). They report *validity*, not *value*: none flags an asset as "no longer meaningful." That semantic judgment is the gap this skill fills, composed **on top of** the structural layer, not instead of it.

## Requirements

- Claude Code with the **Glob**, **Grep**, **Read**, and **Bash** tools. The audit runs in one main context — no subagents required.
- `git` and standard shell (`grep`/`find`) for the inline reachability scan.

## References

The two-stage binary-question design (screen → verdict pressure-test, holistic verdict, no score aggregation) follows the checklist-decomposition evaluation line:

- BinEval "Ask, Don't Judge" — [arXiv:2606.27226](https://arxiv.org/abs/2606.27226)
- CheckEval — [arXiv:2403.18771](https://arxiv.org/abs/2403.18771)
- TICK — [arXiv:2410.03608](https://arxiv.org/abs/2410.03608)

## About this skill

This skill implements the **Maintain** phase of the [Agent Knowledge Cycle (AKC)](https://github.com/shimo4228/agent-knowledge-cycle) — a Zenodo-citable six-phase bidirectional growth loop ([DOI 10.5281/zenodo.19200726](https://doi.org/10.5281/zenodo.19200726)) for sustaining intent alignment between an AI agent and its operator over time. It is a sibling of [`skill-stocktake`](https://github.com/shimo4228/skill-stocktake) (audits installed skills) and [`rules-stocktake`](https://github.com/shimo4228/rules-stocktake) (audits always-loaded rules) — the same stocktake pattern applied to a project repo's non-code assets.

## License

MIT
