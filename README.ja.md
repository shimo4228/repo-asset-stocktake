言語: [English](README.md) | 日本語

# repo-asset-stocktake

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/shimo4228/repo-asset-stocktake)

プロジェクトリポジトリの**非コード資産**（ツール設定・CI/GitHub workflow・runbook その他の docs）に、まだ置いておく価値があるかを棚卸しし、それぞれに **Keep / Update / Retire / Merge** の判定を付ける、Claude Code 向けの [Agent Skill](https://agentskills.io/specification)（英語）です。

linter が見るのは、資産が*構造的に妥当か*（YAML が正しい形か、リンクが生きているか）です。このスキルが問うのは、linter には出せない問い、**この資産はまだ置いておく価値があるか**です。もう走らない `.textlintrc`、発火しても何もしない workflow、数ヶ月前に終わったプロセスを書いた runbook は、どれも全 linter を通過し、どれも重りになっています。このスキルは MegaLinter・actionlint・repolinter のような構造 linter の代わりではなく、その上に重ねて使います。構造の土台（正しい YAML、切れたリンク、使われない依存）は引き続きそちらが受け持ちます。

監査した資産を変えるのは、その資産の判定をあなたが確認したあとだけです。Merge を確認したときは、統合先の資産にも手を入れます。Retire はファイルを消さない soft-delete で、`<file>.disabled` に名前を変えるか、リポジトリ内のゴミ箱に移します。確認なしに書くファイルは 1 つだけで、監査したリポジトリのルートに置く台帳 `.repo-asset-stocktake.json` です。判定と理由、監査の日時を、後の `changed` のために残します。commit したくない場合は ignore ファイルに足すよう、スキルが提案します。

## インストール

```bash
git clone https://github.com/shimo4228/repo-asset-stocktake
mkdir -p ~/.claude/skills
cp -r repo-asset-stocktake/skills/repo-asset-stocktake ~/.claude/skills/repo-asset-stocktake
```

必要なのは、**Glob** / **Grep** / **Read** / **Write** / **Bash** ツールが使える Claude Code と、到達性を調べるための `git` と標準シェル（`grep`/`find`）です。監査は 1 つのメインコンテキストで走り、スキルはサブエージェントも同梱スクリプトもキーも使いません。

repo-asset-stocktake だけを使うなら、このリポジトリを clone してください。著者の [Agent Knowledge Cycle](https://github.com/shimo4228/agent-knowledge-cycle)（英語）のほかのスキルも使うなら、代わりに Claude Code プラグイン [akc-cycle](https://github.com/shimo4228/akc-cycle)（英語）を入れてください。プラグインでは同じスキルを `/akc-cycle:repo-asset-stocktake` という名前で呼びます。このリポジトリは同じ元から一方向に同期しているので、同期と同期の間はプラグインより古いことがあります。

```
/plugin marketplace add shimo4228/akc-cycle
/plugin install akc-cycle@akc-cycle
```

## 使い方

```
/repo-asset-stocktake                    # 現在の repo を監査（既定の full）
/repo-asset-stocktake full /path/to/repo # 別 repo を監査
/repo-asset-stocktake changed            # 変更後の再監査
```

`changed` でも到達性はすべての資産について調べ直し、前回以降に変更された資産と、到達性が変わった資産を判断し直します。ファイル自体に手が入っていなくても、到達性が変わればその対象です。

「この workflow や runbook でもう死んでいるものはある？」のように、ふつうの言葉で頼むこともできます。

1 回の実行は、対応の要るものから並べた判定表 1 つと、Keep・Update・Retire・Merge の件数の 1 行で終わります。表の行は次のようになります（スキルの指示にある例の行で、実際の実行の記録ではありません）。

| Asset | Consumer | Reachability | Verdict | Reason |
|---|---|---|---|---|
| `.textlintrc` | tool-invocation | 0 invocation sites | Retire | textlint dropped from CI and package.json; config now inert |

著者の iOS アプリのリポジトリで最初に回したときは、入口の CLAUDE.md から届かない docs が 33 件見つかり、直し方は削除ではなくリンク 1 本でした（[記事](#著者のほかの仕事)）。

## コアの発想: あらゆる資産には consumer がいる

非コード資産は、存在するから生きているのではありません。何かが*消費する*から生きています。

| consumer | 資産の例 | 死ぬのは… |
|---|---|---|
| **ツール起動**（build script / pre-commit / CI） | `.textlintrc`, `.eslintrc` | どこからもツールが起動されなくなったとき |
| **CI トリガー / runner** | `.github/workflows/*.yml` | 参照先が消えた、またはトリガーが到達不能になったとき |
| **人間の読者**（リンク経由で到達） | runbook, `docs/**` | 誰もリンクしなくなったとき、または終わったプロセスを書いているとき |

設定・workflow・runbook は、上の 3 つの consumer に 1 つずつ対応する、出発点として置いた 3 つの資産の種類です。consumer を名指しすれば、ほかの資産（issue template、governance ファイル、dependabot 設定）にも同じ考え方を広げられます。

## 仕組み: 2 層（tier）、コードで列挙し、判断は LLM に

列挙と判断を分ける設計です。tier 1 は到達性を測ります。これは構造の問題で、grep/find で決まります。tier 2 は価値を判断します。これは意味の問題で、判断が要ります。

1. **Phase 1、棚卸しと tier-1 の到達性**: 資産を列挙し、consumer ごとの到達性をその場の `grep`/`find` で測ります。外部 linter も実行時のスクリプトも使いません。ツールを起動している箇所、workflow の参照先とトリガーの実在、doc への被リンクを調べます。
2. **Phase 2、評価（tier-2）**: yes/no の質問を 2 回に分けて行います。Stage 1 は No があった資産だけを挙げます。Stage 2 は、Keep 以外の判定を覆せないかを問う質問で確かめます。到達性は総合判断の証拠であって、スコアではありません。到達できても死んでいる資産があり、それを見分けるのがこの段です。
3. **Phase 3、まとめ**: 資産ごとの判定表です。理由はそれだけで読めるように書きます。
4. **Phase 4、整理**: Keep 以外の候補は**1 件ずつ**確認します。証拠を先に示してから `[y/n/skip]` を聞き、一括承認はしません。`skip` は判定を記録するだけで、何も変えません。`y` のとき、Retire は **soft-delete**（`.disabled` への名前の変更か、リポジトリ内のゴミ箱への移動）で、自動で完全に削除することはしません。Update は機械的な修正（トリガーを直す、切れた参照を直す、古い設定キーを消す）だけを適用し、文章の書き直しはあなたに戻します。Merge は内容を残る側の資産にまとめてから、吸収された側を soft-delete します。

## 判定の基準

| Verdict | 意味 |
|---|---|
| **Keep** | consumer が生きていて、内容にも意味がある |
| **Update** | consumer は生きているが、内容が古いか一部壊れている。更新する、トリガーを直す、切れた参照を直す |
| **Retire** | consumer が消えたか、内容が形だけになっている。もう置いておく価値がない |
| **Merge into [X]** | 別の資産に置き換わった、または重複している |

## 参考文献

2 回に分けた yes/no の質問（選別、判定を覆せないかの確認、総合判断、スコアに集計しない）は、チェックリストに分解して評価する研究の系列に沿っています。

- BinEval "Ask, Don't Judge": [arXiv:2606.27226](https://arxiv.org/abs/2606.27226)
- CheckEval: [arXiv:2403.18771](https://arxiv.org/abs/2403.18771)
- TICK: [arXiv:2410.03608](https://arxiv.org/abs/2410.03608)

## 著者のほかの仕事

- **[非コード資産の「価値」は linter では測れない —— LLM で棚卸しするスキルを作った](https://zenn.dev/shimo4228/articles/non-code-asset-value-stocktake)**（[English](https://dev.to/shimo4228/linters-cant-measure-a-non-code-assets-value-i-built-an-llm-stocktake-for-it-4ng7)）: 著者の別のリポジトリで最初に回したとき、入口から到達できない docs が 33 件見つかり、削除ではなくリンク 1 本で直した話です。スキルを使わずに同じやり方を回す手順も書いています。
- **[akc-cycle](https://github.com/shimo4228/akc-cycle)**: このスキルをサイクルのほかのスキルと一緒に、1 つの Claude Code プラグインとして入れます（英語）。
- **[Agent Knowledge Cycle](https://github.com/shimo4228/agent-knowledge-cycle)**: サイクルの各フェーズがなぜあるのかを、日付付きの設計判断として記録しています（英語）。このスキルは Maintain（保守）フェーズに属します。リポジトリに溜まったものを、いまもそれを使っているものに照らして見直すフェーズです。
- **[context-sync](https://github.com/shimo4228/context-sync)**: Maintain のスキルの 1 つで、runbook のところでこのスキルと受け持ちが重なります。doc にまだ価値があるかではなく、各事実が正しい文書（CLAUDE.md・ADR・README・graph.jsonld）に置かれ、コードと合っているかを確かめます（英語）。
- **[skill-stocktake](https://github.com/shimo4228/skill-stocktake)**: 同じ棚卸しのやり方をインストール済みのスキルに当て、スキルごとに判定を出します（英語）。
- **[shimo4228](https://github.com/shimo4228/shimo4228)**: 著者の hub です。AKC をほかの長期プロジェクトとその DOI と並べています。

## ライセンス

MIT

<details>
<summary>ツールと AI アシスタント向けの資料</summary>

repo-asset-stocktake は、プロジェクトリポジトリの非コード資産（ツール設定・CI workflow・runbook その他の docs）を監査し、それぞれに Keep・Update・Retire・Merge into [X] のどれかの判定を付ける Claude Code 向けの Agent Skill です。どの linter も通るのに、もう何にも使われていないファイルが溜まっていくリポジトリを保守する人向けです。

このスキルがあるのは、linter が報告するのは妥当性であって価値ではないからです。非コード資産が生きているのは、何かがそれを消費しているからです。ツールの起動、CI のトリガー、リンクをたどって来る人間の読者のどれかです。スキルはその consumer への到達性をコード（grep/find）で測り、到達できる資産にまだ意味があるかを LLM が判断します。到達できても死んでいる資産があるからです（発火しても何もしない workflow、リンクはあるが終わったプロセスを書いた runbook）。到達性がゼロの資産は、少なくとも Retire の候補として必ず挙げます。

基本情報: ライセンスは MIT です。スキル本体（`skills/repo-asset-stocktake/`）はスクリプトを持たない `SKILL.md` 1 つで（版は frontmatter にあります）、著者 1 人（@shimo4228）が保守しています。状態は現役で、著者の Claude Code harness から `scripts/sync-from-local.sh`（commit はしません）で一方向に同期しています。akc-cycle プラグインにも `/akc-cycle:repo-asset-stocktake` として入っているので、同期と同期の間はこのリポジトリがプラグインより古いことがあります。必要なものは Glob・Grep・Read・Write・Bash が使える Claude Code と、git と POSIX シェルです。キーもサブエージェントも要りません。モードは `full`（既定）と `changed` で、リポジトリのパスを付けることもできます。Keep 以外の候補は 1 件ずつ `[y/n/skip]` で確認し、Retire は `<file>.disabled` への名前の変更かリポジトリ内のゴミ箱への移動で行い、Update は機械的な修正（壊れたトリガー、切れた参照）だけを適用して文章の書き直しは人に渡し、判定の記録を監査したリポジトリの `.repo-asset-stocktake.json` に書きます。dead code とコードが読み込むデータファイルは対象外です。

例: まとめの表の 1 行は `.textlintrc | tool-invocation | 0 invocation sites | Retire | textlint dropped from CI and package.json; config now inert` のようになり、最後に資産の総数と、Keep・Update・Retire・Merge それぞれの件数、前回の監査からの変化を 1 行で示します。著者の iOS アプリのリポジトリで最初に回したときは、plan と report のファイル 31 件が、入口の CLAUDE.md からリンクされていない年表のページを通してしか到達できない状態で、そのページと索引を含めて 33 件でした。直し方は削除ではなく、リンク 1 本でした。

リンク: [skills/repo-asset-stocktake/SKILL.md](skills/repo-asset-stocktake/SKILL.md) がスキル本体（英語）、[CHANGELOG.md](CHANGELOG.md) がリリース履歴（英語）、[llms.txt](llms.txt) と [llms-full.txt](llms-full.txt) が機械可読の要約と参照資料（英語）です。このスキルは [Agent Knowledge Cycle (AKC)](https://github.com/shimo4228/agent-knowledge-cycle) の Maintain フェーズの一部を実装しています。AKC の concept DOI（常に最新版へつながる代表 DOI）は [10.5281/zenodo.19200726](https://doi.org/10.5281/zenodo.19200726) で、引用はこの DOI で行います。サイクル全体を入れる形は [akc-cycle](https://github.com/shimo4228/akc-cycle) です。

</details>
