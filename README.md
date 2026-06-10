# Claude Code ガイド集

Claude Code の各機能を、Anthropic 公式ドキュメントと実動作の検証に基づいて**日本語で解説するドキュメント集**です。並列開発・自動化・Agent Teams といったテーマごとにガイドを分けています。

> 各ガイドは執筆時点の公式ドキュメントを照合して書かれています。Claude Code の機能は更新されるため、時間に依存する記述は各ガイド冒頭の**照合日**と最新の公式ドキュメントを併せて確認してください。

## ガイド一覧

### 並列開発

- **[Parallel/GUIDE.md](Parallel/GUIDE.md)** — 複数の Claude Code セッションを Git worktree で同時に走らせ、開発スピードを上げるガイド。worktree の基本操作・競合を防ぐ設計・Todo アプリの並列開発例・トラブルシューティングまでを網羅。公式の `claude --worktree` 機能も別節で解説。（最新照合: 2026年5月24日）

### 自動化（Routines / `/loop` / ワークフローループ）

3 本セット。`GUIDE.md` が本体で、残り 2 本が特定テーマを深掘りする補助ガイドです。

- **[Automation/GUIDE.md](Automation/GUIDE.md)** — **Routines**・**Desktop scheduled tasks**・**`/loop`** を実務でどう使い分けるかの本体ガイド。（最新照合: 2026年5月20日）
- **[Automation/LOOP_GUIDE.md](Automation/LOOP_GUIDE.md)** — 引数なし `/loop` が実行する既定プロンプトを置き換える `.claude/loop.md` の書き方を、独立した実務ガイドとして掘り下げたもの。（最新照合: 2026年6月8日）
- **[Automation/WORKFLOW_LOOP_GUIDE.md](Automation/WORKFLOW_LOOP_GUIDE.md)** — `CLAUDE.md`・Skills・`/loop`・`/goal`・Stop フック・レビューを束ねた、ワークフローループ全体の設計ガイド。（最新照合: 2026年6月9日）

### Agent Teams

- **[Agent_Teams/GUIDE.md](Agent_Teams/GUIDE.md)** — Agent Teams（research preview）の利用ガイド。記述は検証ステータス（✅ 実動作確認済み ｜ 📖 公式ドキュメント記載の例 ｜ ⚠️ 未検証）付きで、確からしさを明示しています。

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
└── .claude/
    ├── commands/agent-team.md     # /agent-team コマンド本体
    └── examples/                  # /agent-team が参照する出力例ライブラリ
```

## 想定読者

Claude Code をすでに使っていて、並列開発・自動化・マルチエージェントといった一歩進んだ使い方を取り入れたい人。
