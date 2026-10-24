Language: [English](README.md) | 日本語

# search-first

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/shimo4228/search-first)

作る前にまず外を調べさせる、Claude Code 向けの [Agent Skill](https://agentskills.io/specification) です。ライブラリを選ぶとき、先行実装を探すとき、論文や公式仕様が実際に何を言っているかを確かめるとき、カスタムコードを書こうとするときに、エージェントがレジストリ・コード・論文・公式ドキュメントを調べ、日付と出典の付いた報告を書きます。あなた（または調査を頼んだセッション）はその報告から選びます。候補をそのまま採るか（`Adopt`）、候補の一部だけを採るか（`Adopt-part`）、どの候補も埋めない部分を自作するか（`Build`）です。

このスキルはエージェント自身の web ツールで検索とページ取得を行うので、検索語はマシンの外へ出ます。スキル独自の API キーは要りません。著者のほかの仕事は、このページ末尾の[一覧](#著者のほかの仕事)にまとめています。

## インストール

単体のスキルとして入れる場合:

```bash
git clone https://github.com/shimo4228/search-first.git
mkdir -p ~/.claude/skills
cp -r search-first/skills/search-first ~/.claude/skills/
```

そのあとは「これに使えるライブラリはある？」のように普通の言葉で頼むか、`/search-first` と入力します。明らかな要件 1 つをエージェントがその場で調べる Quick モードは、これだけで動きます。大きな問いに使う Full モードには、このリポジトリに入っていないサブエージェントがもう 1 つ要ります（[仕組み](#仕組み)を参照）。

同じスキルは、Agent Knowledge Cycle（AKC、コーディングエージェントが繰り返した経験をスキルとルールに変える、著者の 6 フェーズのループ）のほかのスキルと一緒に Claude Code プラグイン [akc-cycle](https://github.com/shimo4228/akc-cycle)（英語）にも入っています。プラグインでは `/akc-cycle:search-first` という名前で呼びます。このリポジトリは同じ元から一方向に同期しているので、同期と同期の間はプラグインより古いことがあります。この段階だけが欲しいなら単体のスキルを、サイクル全体か最新の版が欲しいならプラグインを入れてください。

```
/plugin marketplace add shimo4228/akc-cycle
/plugin install akc-cycle@akc-cycle
```

## 使用例

報告の例です（形式を示すためのもので、各項目は使う前に最新の状態を確かめてください）:

```
## Scope searched
"markdown dead link checker" / "link check CI" — npm, crates.io, GitHub code search — 2026-09-14.
Not located: a checker that resolves wiki-style [[links]] in Obsidian vaults.
## Found
1. lychee (github.com/lycheeverse/lychee) — Rust CLI, checks http and local file links,
   `--offline` mode; last release 2026-08 (read 2026-09-14: README + docs/usage.md). Assumes plain
   markdown links; our vault uses [[wikilinks]], so the http check transfers, the local
   resolution does not.
2. textlint-rule-no-dead-link (github.com/textlint-rule/textlint-rule-no-dead-link) —
   runs inside textlint, http only (saw 2026-09-14: README). Transfers only if textlint is already in
   the pipeline; it is not in ours.
## Verdict
http links: Adopt lychee. [[wikilinks]]: Adopt lychee if it accepts a custom resolver;
otherwise Build — lychee needs its resolver patched (stop 1), textlint-rule-no-dead-link
covers http only; we take on a ~100-line resolver script.
## Still unknown
Whether either tool can be pointed at a custom link resolver.
```

## 仕組み

**外を見て、既製品を採るのが既定。自作には理由を書かせる。** 調べる側は、外に何があり、それがこちらの状況とどう違うかを報告し、要件ごとに判定を書きます。既定は Adopt です。要件を満たす定評ある道具（ライブラリ・CLI・MCP サーバ・プラグイン・公開スキル・エージェント自身の機能）は、多少合わなくても使います。自作したものは自分で保守し続けることになるからです。Build と判定するときは、候補ごとに外した停止条件（内部を fork や patch しないと使えない、つなぎのコードが自作と同じくらい大きい、依存受け入れの手順で落ちる、計画の目的そのものを担わせることになる）と、自作で抱えるものを書きます。どちらかが欠けた Build は Adopt と読みます。計画のための調査では、目的でなく手段にあたる部品（パーサ、CLI のラッパ、eval の runner など）を並べ、部品ごとに道具を問います。Quick モード（後述）では 1 つのエージェントが調べて判定まで書き、あなたはチャットでその判定を覆せます。

1. **問いを文章にする。** 最初のツール呼び出しの前に、エージェントはチャットに 2〜3 文を書きます。問い、その種類、制約（言語・既存の依存・ライセンス・予算）です。これがあるので、手間をかける前にあなたが軌道修正できます。
2. **まずこのリポジトリを調べる。** `rg` / Glob で 1 回だけ検索します。探しているものが既にここにあるかもしれません。
3. **問いの種類ごとに調べる。** 種類ごとに、見る場所と止めどきが決まっています:

   | 問いの種類 | 見る場所 | 止めどき |
   |---|---|---|
   | ライブラリ・ツールの選定 | レジストリ（npm / PyPI / crates）、公式ドキュメント、リリースページ | 2〜3 の候補について一次資料の事実がそろった |
   | 先行実装 | GitHub のコード検索、skill / agent / MCP のカタログ、このリポジトリ | 実物のファイルを 2 つ読んだ、または独立した 3 つの情報源が一致した |
   | 論文・記事の主張 | 一次資料の本文（arXiv、公式ブログ、リポジトリのファイルそのもの） | 対象と前提を書き出せた |
   | 仕様・公式の挙動 | 公式ドキュメント、`--help`、changelog、実際に動かした結果 | 版の明記された一次資料の事実が 1 つ得られた |
   | 実務の現状 | 上の行の組み合わせと、実際に運用した人の記録 | 次の検索で決着する問いをもう挙げられない |
   | 一次資料の確認 | 主張が指している本文（要旨の先まで） | すべての主張に、裏付けあり・一部・裏付けなし・未確認のどれかが付いた |

4. **報告する。** すべての主張に調べた時点の日付と出典 URL を付け、本文まで読んだのか、検索結果の抜粋だけを見たのかを書きます（上の例の、日付を添えた `read` / `saw`）。何も見つからなかったときは調べた範囲を書くので、次のセッションはそこから範囲を広げられます。評価は事実に裏付けた文章で書き、**数値のスコアや等級は付けません**。まだ分からないことを最後に挙げる節の前に Verdict の節を置き、要件ごとに 1 つの判定を書きます。`Adopt <候補>`、`Adopt-part <候補: 採る部分>`、またはどの候補も埋めない穴を添えた `Build` です。計画のための調査では、調査の報告には Verdict を書かず、計画そのものに書きます。

明らかな要件が 1 つだけなら、エージェントはこの手順をその場で回します（Quick）。単純でない問いや、計画のために頼まれた問いでは、問いを 3〜5 の角度（うち 1 つは反証を探す角度）に分け、角度ごとに `researcher` サブエージェント（1 つの角度を調べてメモを書く、別の Claude Code エージェント）を 1 つずつ起動します（Full）。各サブエージェントの予算は検索とページ取得あわせて 10〜15 回です。`researcher` サブエージェントは、このリポジトリにも akc-cycle プラグインにも入っていません。著者のものを使うなら、claude-harness の [`agents/researcher.md`](https://github.com/shimo4228/claude-harness/blob/main/agents/researcher.md) を `~/.claude/agents/` にコピーします。著者の harness ではフック（`research-gate.sh`）がこのエージェントの書き込み先も制限していますが、ファイルをコピーしてもこのフックは付いてきません。

あなたが「調べなくていい」と言ったときは、エージェントはそのことを 1 行記録してから進めます。その 1 行の最後には短い調査の申し出があり、作り始める前ならその調査を頼むこともできます。

## トリガー条件

- ライブラリやツールを選ぶとき、依存を追加するとき
- ユーティリティ・ヘルパー・アダプタを書く前に先行実装を探すとき
- 論文や記事がこちらに当てはまるかを確かめるとき、主張を出典と照らすとき
- 公式仕様のいまの挙動を読むとき、実務の現状を調べるとき

## 著者のほかの仕事

- **[コーディングエージェントの知識をどこに置き、どう守らせるか](https://zenn.dev/shimo4228/articles/coding-agent-memory-architecture)**（[English](https://dev.to/shimo4228/where-to-put-a-coding-agents-knowledge-and-how-to-make-it-stick-161g)）: このスキルの以前の版を計測すると、プロンプトが検索を後押ししたときでも、エージェントは候補の比較と判断の宣言を飛ばしていました。報告に要件ごとの判定があるのはそのためです。
- **[推論でもツールでもない — AIエージェントの本質は「記憶」ではないか](https://zenn.dev/shimo4228/articles/agent-essence-is-memory)**（[English](https://dev.to/shimo4228/not-reasoning-not-tools-what-if-the-essence-of-ai-agents-is-memory-4k4n)）: search-first・learn-eval・[rules-distill](https://github.com/shimo4228/rules-distill)（複数のスキルに繰り返し出てくる原則をルールにするスキル、英語）が 1 つずつ作られ、エージェントの記憶を回す 1 つのループになっていった経緯です。
- **[akc-cycle](https://github.com/shimo4228/akc-cycle)**: このスキルをサイクルのほかのスキルと一緒に、1 つの Claude Code プラグインとして入れます（英語）。
- **[Agent Knowledge Cycle](https://github.com/shimo4228/agent-knowledge-cycle)**: Research を含むサイクルの各フェーズがなぜあるのかを、日付付きの設計判断として記録しています。
- **[learn-eval](https://github.com/shimo4228/learn-eval)**: 次のフェーズです。セッションで学んだことを、将来のセッションがたどり着く場所に保存します。
- **[signal-first-research](https://github.com/shimo4228/signal-first-research)**: 次の行動を変えそうな情報だけを受け入れるリサーチの取り込みフィルタの設計ノートです。凍結済みで、その原則はいまこのスキルなどに引き継がれています（英語）。
- **[shimo4228](https://github.com/shimo4228/shimo4228)**: 著者の拠点リポジトリです。AKC をほかの長期プロジェクトとその DOI と並べています。

## ライセンス

MIT

<details>
<summary>ツールと AI アシスタント向けの資料</summary>

search-first は、コーディングエージェントに判断の時点で外部の情報源を調べさせ、何があるかを報告させる Agent Skill（オープンな Agent Skills 形式の Markdown の指示ファイル 1 つ）です。既存のライブラリを作り直すのをやめさせ、各主張の出典を示させたい開発者向けです。要件ごとに判定を書き、既定は Adopt です。Build を選ぶのは、候補ごとに外した停止条件を書けたときだけです。

このスキルがあるのは、記憶から作るエージェントは既にあるものを作り直し、古くなったパッケージの事実を信じてしまうからです。定評ある道具があれば多少無理をしてでも採るのが既定で、自作を選ぶのは、どの候補も停止条件に当たったときだけです。以前の版を著者の skill-comply で 2026-03 に 3 つのプロンプトの場面で計測したところ、検索が守られたのはプロンプトがスキルを後押ししたときだけで、そのときでも候補の比較と判断の宣言は飛ばされました（その場面で守られた手順は 40% で、数値は README からリンクしている記事「コーディングエージェントの知識をどこに置き、どう守らせるか」にあります）。要件ごとに Verdict の節を持ついまの報告の形は、その比較を報告の中に書かせるためのものです。

基本情報: ライセンスは MIT です。コードはなく `SKILL.md` 1 つだけで、著者 1 人（@shimo4228）が保守しています。状態は現役で、著者の Claude Code harness から `scripts/sync-from-local.sh`（commit はしません）で一方向に同期しています。akc-cycle プラグインにも入っているので、同期と同期の間はこのリポジトリがプラグインより古いことがあります。必要なものは web 検索とページ取得のツールを持つエージェントで、スキル独自の有料キーはありません。Full モードは、このリポジトリが同梱しない `researcher` サブエージェントを前提にし、メモを `$HOME/.cache/claude-research-notes/` に書きます。計画のための調査では `<repo>/docs/plans/research/` に報告を書きます。Adopt や Adopt-part（パッケージ・MCP サーバ・プラグイン・スキル）は、呼び出し側の依存受け入れの手順を通してから入れる前提です。

例: チャットで返す報告は 4 つの節からなります（計画のための報告は `Verdict` を書かず、計画そのものに書きます）。`Scope searched`（検索語・情報源・調べた時点の日付・見つからなかったもの）、`Found`（項目ごとに URL、対象と前提、証拠の種類、こちらの状況との違い、持ち込める部分）、`Verdict`（要件ごとに `Adopt <候補>`、`Adopt-part <候補: 採る部分>`、または候補ごとの停止条件を添えた `Build`）、`Still unknown` です。評価は日付と版を添えた文章で書き、数値のスコアにはしません。

リンク: [skills/search-first/SKILL.md](skills/search-first/SKILL.md) がスキル本体（英語）、[llms.txt](llms.txt) と [llms-full.txt](llms-full.txt) が機械可読の要約と参照資料（英語）です。このスキルは [Agent Knowledge Cycle (AKC)](https://github.com/shimo4228/agent-knowledge-cycle) の Research フェーズを実装しています。AKC の concept DOI（常に最新版へつながる代表 DOI）は [10.5281/zenodo.19200726](https://doi.org/10.5281/zenodo.19200726) で、引用はこの DOI で行います。サイクル全体を入れる形は [akc-cycle](https://github.com/shimo4228/akc-cycle) です。

</details>
