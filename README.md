# Claude Code ガイド集

Claude Code の各機能を、Anthropic 公式ドキュメント（および一部ガイドでは実動作の検証）に基づいて**日本語で解説するドキュメント集**です。並列開発・自動化・Agent Teams・プロンプティングといったテーマごとにガイドを分けています。

> 各ガイドは執筆時点の公式ドキュメントを照合して書かれており、**照合日は各ガイドの冒頭**に記載しています。Claude Code の機能は更新されるため、時間に依存する記述は照合日と最新の[公式ドキュメント](https://code.claude.com/docs)を併せて確認してください。

## 最新情報

- **2026年6月11日** — Claude Fable 5 プロンプティングガイド [Prompts/CLAUDE_FABLE_5.md](Prompts/CLAUDE_FABLE_5.md) を追加しました。公式ドキュメント「Claude Fable 5 のプロンプティング」の要点と日本語訳に、移行チェックリストと Claude Code への適用レシピ（付録）を加えています。
- **2026年6月10日** — ワークフローループ設計ガイド [Automation/WORKFLOW_LOOP_GUIDE.md](Automation/WORKFLOW_LOOP_GUIDE.md) を追加しました。`CLAUDE.md`・Skills・`/loop`・`/goal`・Stop フック・レビューを束ねた、ワークフローループ全体の設計をまとめています。

## どこから読むか

- **Claude Code を使い始めたばかり / まず並列開発を試したい** → [Parallel/GUIDE.md](Parallel/GUIDE.md)（手順を一つずつ説明する、最も入門寄りのガイド）
- **定期実行・自律ループで作業を自動化したい** → [Automation/GUIDE.md](Automation/GUIDE.md)（自動化3部作の入口）
- **複数エージェントに分担させたい** → [Agent_Teams/GUIDE.md](Agent_Teams/GUIDE.md) と、同梱の `/agent-team` コマンド（下記）
- **Claude Fable 5 に合わせてプロンプトやスキャフォールディングを調整したい** → [Prompts/CLAUDE_FABLE_5.md](Prompts/CLAUDE_FABLE_5.md)

## ガイド一覧

### 並列開発

- **[Parallel/GUIDE.md](Parallel/GUIDE.md)** — 複数の Claude Code セッションを Git worktree で同時に走らせ、開発スピードを上げるガイド。worktree の基本操作・競合を防ぐ設計・Todo アプリの並列開発例・トラブルシューティングまでを網羅。公式の `claude --worktree` 機能も別節で解説。

### 自動化（Routines / `/loop` / ワークフローループ）

3 本セット。`GUIDE.md` が本体で、残り 2 本が特定テーマを深掘りする補助ガイドです。

- **[Automation/GUIDE.md](Automation/GUIDE.md)** — **Routines**・**Desktop scheduled tasks**・**`/loop`** を実務でどう使い分けるかの本体ガイド。
- **[Automation/LOOP_GUIDE.md](Automation/LOOP_GUIDE.md)** — 引数なし `/loop` が実行する既定プロンプトを置き換える `.claude/loop.md` の書き方を、独立した実務ガイドとして掘り下げたもの。
- **[Automation/WORKFLOW_LOOP_GUIDE.md](Automation/WORKFLOW_LOOP_GUIDE.md)** — `CLAUDE.md`・Skills・`/loop`・`/goal`・Stop フック・レビューを束ねた、ワークフローループ全体の設計ガイド。

### Agent Teams

- **[Agent_Teams/GUIDE.md](Agent_Teams/GUIDE.md)** — Agent Teams（research preview）の利用ガイド。記述は検証ステータス（✅ 実動作確認済み ｜ 📖 公式ドキュメント記載の例 ｜ ⚠️ 未検証）付きで、確からしさを明示しています。

### プロンプティング

- **[Prompts/CLAUDE_FABLE_5.md](Prompts/CLAUDE_FABLE_5.md)** — Claude Fable 5 のプロンプティング解説・日本語訳・実践ガイド。公式ドキュメント「Claude Fable 5 のプロンプティング」が示す 14 のパターンとスキャフォールディング変更を要点と日本語訳で整理し、付録として移行チェックリストと Claude Code への適用レシピ（`CLAUDE.md` 階層・hooks・`/effort`・auto memory など）を収録。

## 同梱の `/agent-team` コマンド

`.claude/commands/agent-team.md` は、与えたタスクから複数チームメートが協調するためのプロンプトを設計・生成する **`/agent-team` スラッシュコマンド**です。生成時に `.claude/examples/` の出力例を手本にする設計で、良い出力例を `.claude/examples/` に `.md` で追加するほど生成品質が安定します（運用方針は [`.claude/examples/README.md`](.claude/examples/README.md) を参照）。

## リポジトリ構成

```
.
├── Parallel/GUIDE.md              # 並列開発（Git worktree）
├── Automation/
│   ├── GUIDE.md                   # Routines / Desktop tasks / loop の使い分け（本体）
│   ├── LOOP_GUIDE.md              # .claude/loop.md の書き方（補助）
│   └── WORKFLOW_LOOP_GUIDE.md     # ワークフローループ設計（補助）
├── Agent_Teams/GUIDE.md           # Agent Teams 利用ガイド
├── Prompts/CLAUDE_FABLE_5.md      # Claude Fable 5 プロンプティングガイド
└── .claude/
    ├── commands/agent-team.md     # /agent-team コマンド本体
    └── examples/                  # /agent-team が参照する出力例ライブラリ
```

## 想定読者

各ガイドに共通する前提は、**Claude Code を使い始めた人**で、Git の基本操作（clone・commit・push）とターミナルに触れたことがあるレベルです（コマンドは Mac / Linux 向け）。難易度はガイドにより差があり、[Parallel/GUIDE.md](Parallel/GUIDE.md) が最も入門寄り、自動化・Agent Teams は Claude Code の基本操作に慣れていることを前提とします。[Prompts/CLAUDE_FABLE_5.md](Prompts/CLAUDE_FABLE_5.md) は Claude API でのハーネス構築や hooks の設定にも踏み込むため、最も上級寄りです。

## ライセンス

本リポジトリのガイド・ドキュメントは [Creative Commons Attribution 4.0 International（CC BY 4.0）](https://creativecommons.org/licenses/by/4.0/) の下で公開されています。クレジット（著作者表示）と本ライセンスへのリンクを示せば、改変・再配布・商用利用を含めて自由に利用できます。詳細は [LICENSE](LICENSE) を参照してください。
