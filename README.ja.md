言語: [English](README.md) | 日本語

# repo-asset-stocktake

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/shimo4228/repo-asset-stocktake) [![GitMCP](https://img.shields.io/endpoint?url=https://gitmcp.io/badge/shimo4228/repo-asset-stocktake)](https://gitmcp.io/shimo4228/repo-asset-stocktake)

プロジェクトリポジトリの**非コード資産** — ツール設定・CI/GitHub workflow・runbook その他の docs — を*価値の減衰*で棚卸しし、それぞれに **Keep / Update / Retire / Merge** の verdict を付ける [Agent Skill](https://agentskills.io/specification)。

linter が見るのは資産が*構造的に妥当か*（YAML が well-formed か、リンクが生きているか）。このスキルが問うのは linter に出せない問い — **この資産はまだ存在に値するか**。もう走らない `.textlintrc`、発火するが no-op の workflow、数ヶ月前に廃れたプロセスを書いた runbook — どれも全 linter を通過し、どれも死んだ重り。

## コアの発想 — あらゆる資産には consumer がいる

非コード資産は「存在するから」生きているのではなく、何かが*消費する*から生きている:

| consumer | 資産の例 | 死ぬのは… |
|---|---|---|
| **ツール起動**（build script / pre-commit / CI） | `.textlintrc`, `.eslintrc` | どこからもツールが起動されなくなったとき |
| **CI トリガー / runner** | `.github/workflows/*.yml` | 参照が消える / トリガーが到達不能になったとき |
| **人間の読者**（リンク経由で到達） | runbook, `docs/**` | 誰もリンクせず、廃れたプロセスを記述しているとき |

configs / workflows / runbooks は3つの **seed** consumer クラス — consumer を定義すれば任意の資産（issue template, governance ファイル, dependabot 設定）へ拡張できる。

## インストール

### Claude Code

```bash
cp -r skills/repo-asset-stocktake ~/.claude/skills/repo-asset-stocktake
```

### SkillsMP

```bash
/skills add shimo4228/repo-asset-stocktake
```

## 使い方

```
/repo-asset-stocktake                    # 現在の repo を監査
/repo-asset-stocktake full /path/to/repo # 別 repo を監査
/repo-asset-stocktake changed            # 前回以降の変更のみ再監査
```

## 仕組み — code → judgment の 2 段

**enumerate / decide** 分割に従う: 到達性は構造的（grep/find で決まる）、価値は意味的（判断が要る）。

1. **Phase 1 — Inventory + tier-1 reachability**: 資産を列挙し、consumer ごとの到達性を inline `grep`/`find` で計測。外部 linter なし・runtime script なし。ツール起動サイト、workflow 参照/トリガーの実在、doc の inbound-link グラフ。
2. **Phase 2 — Evaluation (tier-2)**: 2 段の binary screen（Stage 1 は Yes/No、No のみ surface。Stage 2 は反証質問で非 Keep verdict を pressure-test）。到達性は holistic 判断の証拠であってスコアではない。到達可能でも死んでいる資産があり、その判断はここに宿る。
3. **Phase 3 — Summary**: 資産ごとの verdict テーブル（自己完結した理由付き）。
4. **Phase 4 — Consolidation**: 非 Keep 候補は**1件ずつ**確認 — 証拠を先に示し `[y/n/skip]`、一括承認はしない。Retire は**soft-delete 先行**（`.disabled` rename）、自律 hard-delete はしない。

## verdict 基準

| Verdict | 意味 |
|---|---|
| **Keep** | consumer 生存・内容有意 |
| **Update** | consumer 生存だが内容が stale / 一部破損 — refresh・トリガー修復・dead ref 修正 |
| **Retire** | consumer 消失 or 内容形骸化 — もう存在に値しない |
| **Merge into [X]** | 他資産に superseded / 重複 |

## なぜ構造 linter（MegaLinter, actionlint, repolinter）でないのか

それらは優秀で、このスキルは置き換えない — **構造**の床（valid YAML, dead link, unused dep）を担う。だが報告するのは*妥当性*で*価値*ではない。「もう意味がない」と資産を flag するものは無い。その意味的判断がこのスキルの埋める gap で、構造層の**上に**組む（代わりでなく）。

## 要件

- Claude Code の **Glob** / **Grep** / **Read** / **Bash** ツール。監査は単一 main context で走る（subagent 不要）。
- inline 到達性スキャン用の `git` と標準シェル（`grep`/`find`）。

## 参考文献

2 段の binary-question 設計（screen → verdict pressure-test、holistic verdict、score 集約なし）は checklist-decomposition 評価系列に依拠:

- BinEval "Ask, Don't Judge" — [arXiv:2606.27226](https://arxiv.org/abs/2606.27226)
- CheckEval — [arXiv:2403.18771](https://arxiv.org/abs/2403.18771)
- TICK — [arXiv:2410.03608](https://arxiv.org/abs/2410.03608)

## このスキルについて

[Agent Knowledge Cycle (AKC)](https://github.com/shimo4228/agent-knowledge-cycle) の **Maintain** フェーズの実装 — AI エージェントと運用者の意図整合を時間をかけて保つ、Zenodo 引用可能な 6 フェーズ双方向成長ループ（[DOI 10.5281/zenodo.19200726](https://doi.org/10.5281/zenodo.19200726)）。[`skill-stocktake`](https://github.com/shimo4228/skill-stocktake)（インストール済み skill を監査）・[`rules-stocktake`](https://github.com/shimo4228/rules-stocktake)（常駐 rule を監査）の sibling で、同じ stocktake パターンをプロジェクト repo の非コード資産に適用する。

## ライセンス

MIT
