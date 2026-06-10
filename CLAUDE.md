# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## このリポジトリの性格

Claude Code の各機能（並列開発・自動化・Agent Teams など）を**日本語で解説するドキュメント集**。ソフトウェアプロジェクトではないため、ビルド・リント・テストのツールチェーンやパッケージマニフェストは存在しない。成果物は Markdown ガイドそのもので、ここでの「作業」はほぼ Markdown の執筆・編集・相互参照の保守を指す。

## 全体構成（トピック別ディレクトリ）

ガイドはトピックごとにディレクトリへ分かれ、各ディレクトリの中心ファイルは `GUIDE.md`。

- **`Parallel/GUIDE.md`** — Git worktree で複数 Claude Code セッションを並列に走らせる開発ガイド。`git worktree` を直接扱う流儀が中心で、公式の `claude --worktree` 機能は専用節にまとめてある。冒頭に自己参照の目次（見出しアンカー）を持つ唯一のガイド。
- **`Automation/`** — `/loop`・Routines・Desktop scheduled tasks・ワークフローループの 3 本セット。**親子関係**になっている（下記）。
  - `GUIDE.md`（**親**）— Routines / Desktop scheduled tasks / `/loop` の使い分け本体。§番号で章立てされ、他 2 本から §4 などとして参照される。
  - `LOOP_GUIDE.md`（補助）— `.claude/loop.md` の書き方を親ガイド §4 から抜き出して深掘りしたもの。
  - `WORKFLOW_LOOP_GUIDE.md`（補助）— `CLAUDE.md`・Skills・`/loop`・`/goal`・Stop フック・レビューを束ねたワークフローループ全体の設計図。
- **`Agent_Teams/GUIDE.md`** — Agent Teams（research preview）の利用ガイド。
- **`.claude/commands/agent-team.md`** — `/agent-team` スラッシュコマンドの実体（下記のメタプロンプト）。
- **`.claude/examples/`** — `/agent-team` が参照する出力例ライブラリ。

## 把握しておくべき横断構造

単一ファイルを読むだけでは見えない、複数ファイルにまたがる関係が 2 つある。

### 1. Automation 3 ガイドの親子・相互参照

`GUIDE.md` が親で、`LOOP_GUIDE.md` と `WORKFLOW_LOOP_GUIDE.md` がその特定章を深掘りする補助ガイド。3 本は本文中で互いを**リポジトリルートからの相対パス**（例 `` `Automation/LOOP_GUIDE.md` ``）と**親ガイドの §番号**（例 `GUIDE.md §4`）で指し合う。いずれかを編集する際は、参照しているファイル名・§番号がずれていないかを必ず合わせて確認する。

### 2. `/agent-team` ↔ `.claude/examples/` のフィードバックループ

`.claude/commands/agent-team.md` は、ユーザーのタスクからエージェントチーム用プロンプトを設計・生成するメタプロンプト（生成物は `.claude/agent-team-plan.md` に保存しユーザー確認を取る）。生成時に **Step 6 で `.claude/examples/` を読み、出力スタイルの手本にする**。`.claude/examples/README.md` が運用方針を定めており、「良い出力例を examples に `.md` で追加するほど生成品質が安定する」設計。examples を増やす／改名する際はこの参照関係を壊さないこと。

## 執筆規約（既存ガイドに共通、編集時は踏襲する）

- **照合日の明記**: 各ガイドは冒頭で「YYYY年MM月DD日時点の Anthropic 公式ドキュメントを照合」と宣言し、時間に依存する事実（機能の挙動・制限・research preview 状態など）には照合日を併記する。Claude Code の機能は変わるため、事実を更新したら照合日も更新する。
- **検証ステータスの凡例**（`Agent_Teams/GUIDE.md` で使用）: ✅ 実動作確認済み ｜ 📖 公式ドキュメント記載の例 ｜ ⚠️ 未検証（推定）。記述の確からしさをこの記号で明示する。
- **相互参照はリポジトリルートからの相対パス**で書く。`references/` のような接頭辞は付けない（過去に混在していたが現構成に合わせて除去済み）。
- **全文日本語**。コマンド名・パス・英語の技術用語はバッククォートで囲む。

## リンク切れ・相互参照の確認

ガイド同士の相互参照が多いため、ファイルを移動・改名したらルート相対パスが実在するか確認する。

```bash
# 全 Markdown から *.md 参照を抽出して棚卸し（説明用の .claude/... 等を含むので目視で選別）
grep -rhoE "[A-Za-z0-9_./<>-]+\.md" --include="*.md" . | sort | uniq -c | sort -rn

# 想定外の接頭辞が残っていないか（references/ は使わない方針）
grep -rn "references/" --include="*.md" .
```

なお `.claude/loop.md`・`~/.claude/CLAUDE.md`・`.claude/skills/<name>/SKILL.md` のような記述は、**読者が自分の環境で作るファイルの説明**であり、このリポジトリ内に存在しなくて正常。リポジトリ内の実ファイルを指す相互参照（`Automation/*.md` など）とは区別する。
