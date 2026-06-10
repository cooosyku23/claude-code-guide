# Claude Code ワークフローループ設計ガイド

Claude Codeで、単発プロンプトではなく、**エージェントが継続的に作業・検証・修正・レビューを回すワークフローループ** をどう設計するかを整理します。本ガイドは `Automation/GUIDE.md` の補助ガイドで、`GUIDE.md §4`（`/loop`・Skills・`/goal`）と `LOOP_GUIDE.md`（`.claude/loop.md` の書き方）を、ワークフローループ全体の設計という観点から束ねます。

本ガイドの技術的事実は 2026-06-09 に Claude Code 公式ドキュメントと照合しています。挙動・制限は変更されうるため、時間に依存する事実には照合日を併記します。

主な一次情報:

- [How Claude Code works](https://code.claude.com/docs/en/how-claude-code-works)
- [Commands](https://code.claude.com/docs/en/commands)
- [Best practices for Claude Code](https://code.claude.com/docs/en/best-practices)
- [Extend Claude with skills](https://code.claude.com/docs/en/skills)
- [Run prompts on a schedule](https://code.claude.com/docs/en/scheduled-tasks)
- [Keep working toward a goal](https://code.claude.com/docs/en/goal)
- [Automate actions with hooks](https://code.claude.com/docs/en/hooks-guide)
- [Hooks reference](https://code.claude.com/docs/en/hooks)
- [Settings](https://code.claude.com/docs/en/settings)

関連ガイド: `/loop`・Routines・Desktop scheduled tasks の使い分けは `Automation/GUIDE.md`、`.claude/loop.md` の詳しい書き方は `Automation/LOOP_GUIDE.md` を参照してください。

---

## クイックスタート: まず何を打つか

「結局どれを打てばループが始まるのか」を最初に決めます。セッション内でループを始める入口は実質2つで、`/goal`（条件を満たすまで自動で継続。[Keep working toward a goal](https://code.claude.com/docs/en/goal)）と `/loop`（一定間隔でプロンプトを繰り返す。[Run prompts on a schedule](https://code.claude.com/docs/en/scheduled-tasks)）です。どちらも1回打てば、あとは自動で反復します。残りの部品（`CLAUDE.md`・Stop フック・スキル）は「開始するもの」ではなく、事前の土台や1回呼び出す手順です。

```text
何をしたい？
  │
  ├─ この作業を完了条件までやり切らせたい（例: テストが通るまで実装）
  │     → /goal（条件ベース。条件を満たすまでターンを跨いで自動継続し、満たしたら止まる）
  │
  └─ 一定間隔で同じ確認・保守を繰り返したい（例: CI・PR・デプロイの監視）
        → /loop（時間ベース。プロンプトを間隔ごとに再実行する）
```

実際に打つコマンド例は §5「実行例」、その前提となる最小構成（必要なファイルと順序）は §4.1「実用的な最小構成」にあります。`/goal` と `/loop` の使い分け・詳しい挙動は §4.5、Stop フックでの機械的な強制は §4.6 を参照してください。

---

## 1. 概要

Claude Codeでワークフローループを設計するなら、中心となるのは次の構成要素です。

1. **`CLAUDE.md`**：プロジェクトの恒久的な前提・ルールを書く
2. **`.claude/skills/.../SKILL.md`**：再利用可能な作業手順を `/skill-name` として定義する
3. **`.claude/loop.md` と `/loop`**：現在のセッション内で、定期的に同じ保守・確認プロンプトを回す
4. **`/goal` / Stop フック**：時間間隔ではなく「条件を満たすまで」作業を継続させる（`/goal` はモデル側で継続を促し、Stop フックは停止を機械的にブロックする。役割が異なり、併用もできる）
5. **フック / テスト / lint / 型チェック / レビュー用サブエージェント**：停止条件と品質ゲートを機械化する

重要なのは、`/loop` だけで全部をやろうとしないことです。`/loop` は「一定間隔で再実行する仕組み」、`/goal` や Stop フックは「条件を満たすまで止めない仕組み」、スキルは「手順の再利用」、フックは「必ず実行される安全装置」です。

TDD を例に挙げると、まずは **Red → Green → Refactor → テスト → レビュー → 修正 → 完了** を `.claude/skills/tdd-loop/SKILL.md` に定義し、そのうえで `.claude/loop.md` には「現在ブランチのPR・CI・失敗テストを見て、必要ならそのスキル相当の手順を回す」と書くのが現実的です。

## 2. Claude Codeにおける “ループ設計” のレイヤーを分ける

Claude Code のエージェントループ自体は、Claude がファイルを読み、コマンドを実行し、結果を見て次の判断をする内部ループです。Claude Code 公式ドキュメントでも、Claude Code はモデル・ツール・実行環境を提供するエージェント実行基盤であり、ツール実行結果が次の判断にフィードバックされると説明されています。([How Claude Code works](https://code.claude.com/docs/en/how-claude-code-works))

ユーザーが設計するのは、その上に乗る **ワークフローループ** です。

| レイヤー     | 目的                          | Claude Codeで使うもの              |
| -------- | --------------------------- | ----------------------------- |
| 作業方針     | 毎回守る前提を与える                  | `CLAUDE.md`                   |
| 手順       | 反復可能な作業をコマンド化する             | `.claude/skills/.../SKILL.md` |
| 定期実行     | 数分おき・数十分おきに再確認する            | `/loop`, `.claude/loop.md`    |
| 条件達成まで継続 | テストがグリーンになるまで止めない           | `/goal`, Stop フック            |
| 機械的保証    | フォーマッタ・lint・テスト・禁止操作        | フック                           |
| 独立レビュー   | 実装者とは別の視点で確認                | サブエージェント, `/code-review`      |

Claude Codeでは `/loop` は同梱スキルとして提供されており、セッションを開いたままプロンプトを繰り返し実行できます。間隔を省略すると Claude が自己調整し、プロンプトを省略すると組み込みのメンテナンスプロンプトまたは `.claude/loop.md` が使われます。([Commands](https://code.claude.com/docs/en/commands))

なお、本稿で扱う機能には最低バージョンがあります。`/loop`（scheduled tasks）は Claude Code v2.1.72 以降、`/goal` は v2.1.139 以降が必要です（2026-06-09 時点）。これより古いバージョンではコマンド自体が存在せず、本稿の構成は動きません。

---

## 3. ワークフローループの処理の流れ

まず、ループをこういう形で設計します。

```text
入力
  ↓
コンテキスト収集
  ↓
次のアクションを計画・判断
  ↓
実装または修正
  ↓
検証を実行
  ↓
差分をレビュー
  ↓
条件を満たせば停止
  ↓
満たさなければ継続
```

TDDの場合、具体的にはこうです。

```text
失敗テストを確認
  ↓
失敗理由を診断
  ↓
最小実装
  ↓
該当テストを再実行
  ↓
全体テスト・lint・型チェック
  ↓
差分レビュー
  ↓
問題がなければ終了
  ↓
問題があれば修正に戻る
```

Claude Codeのベストプラクティスでも、テスト・ビルド・スクリーンショット比較など、Claude が読み取れる合否の検証手段を与えることが重要だとされています。検証手段があると、Claude は作業し、チェックを実行し、結果を読んで、通るまで反復できます。([Best practices for Claude Code](https://code.claude.com/docs/en/best-practices))

---

## 4. 各構成要素を作って組み合わせる

以下の各構成要素は、§1・§3 と同じく TDD を例として説明します。スキル名（`tdd-loop`）や手順は TDD に即した具体例であり、レビュー・リファクタリング・移行など、検証可能な完了条件を持つ反復作業であれば同じ構成を流用できます。

### 4.1 実用的な最小構成

最初から全部作るより、この順番がよいです。

```text
ステップ1: CLAUDE.md
  ↓
ステップ2: /tdd-loop スキル
  ↓
ステップ3: .claude/loop.md
  ↓
ステップ4: /goal
  ↓
ステップ5: Stop フック
  ↓
ステップ6: /code-review またはサブエージェントレビュー
```

最小構成なら、まずこの3ファイルで十分です。

```text
CLAUDE.md
.claude/skills/tdd-loop/SKILL.md
.claude/loop.md
```

このうち `.claude/loop.md` は §4.4 の裸の `/loop` を使う保守ルート向けで、§5 冒頭の `/goal` ＋ `/tdd-loop`（実装を完了条件まで回すルート）では使いません。実装ループを回すだけなら `CLAUDE.md` と `tdd-loop` スキルの2つで足り、`.claude/loop.md` は `/loop` を併用するときに追加します。

---

### 4.2 `CLAUDE.md` には「恒久ルール」だけを書く

`CLAUDE.md` には、毎回必要な前提を書きます。ここに巨大な手順を書きすぎるより、方針・参照先・禁止事項に絞る方が扱いやすいです。

例：

```md
# プロジェクト指示

## 開発スタイル

- TDD を優先する。
- 実装の前に、失敗しているテストを理解するか、最小の失敗するテストを書く。
- 明示的な承認なしにスコープを広げない。
- 小さく、レビュー可能な変更を優先する。
- 編集後は、まず最も狭い関連テストを実行し、その後に広い範囲のチェックを行う。

## 検証コマンド

- ユニットテスト: `npm test`
- 型チェック: `npm run typecheck`
- lint: `npm run lint`
- フォーマット: `npm run format`

## 安全のために

- 現在の会話で明示的に許可されていない限り、プッシュ・ブランチ削除・履歴の書き換え・破壊的コマンドの実行を行わない。
```

Claude Code のコマンドリファレンスでは、初回セッションで `/init` により `CLAUDE.md` を生成し、`/memory` で調整する流れが紹介されています。([Commands](https://code.claude.com/docs/en/commands))

---

### 4.3 再利用する作業手順はスキルにする

毎回同じTDD手順を貼るなら、`CLAUDE.md` ではなくスキルにするのが向いています。Claude Codeでは、`.claude/skills/` に `SKILL.md` を置くことで、`/skill-name` として直接呼び出せる再利用可能な手順を作れます。公式ドキュメントでも、同じ指示・チェックリスト・複数ステップ手順を繰り返し貼っている場合はスキル化が適していると説明されています。([Extend Claude with skills](https://code.claude.com/docs/en/skills))

例：

```text
.claude/
  skills/
    tdd-loop/
      SKILL.md
```

```md
---
name: tdd-loop
description: テストとレビュー基準が通るまで TDD 実装ループを回す
disable-model-invocation: true
---

# TDD ループ

実装タスクではこのワークフローを使う。

## ループ

1. 要求された挙動と既存の設計を理解する。
2. 最小の失敗するテストを特定するか、書く。
3. 最も狭い関連テストを実行し、期待どおりの理由で失敗することを確認する。
4. テストを通せる最小の変更を実装する。
5. 最も狭いテストを再実行する。
6. より広い検証を実行する:
   - ユニットテスト
   - 型チェック
   - lint
7. `git diff` を確認する。
8. 次をチェックする:
   - スコープクリープ
   - 不要な抽象化
   - 抜けているエッジケース
   - 不足しているテスト
9. 変更が非自明な場合は、新規のサブエージェントまたは `/code-review` で差分をレビューする。
10. 指摘があれば修正し、検証を再実行する。

## 停止してよい条件

- 関連テストが通っている。
- 型チェックが通っている。
- lint が通っている。
- 差分が依頼されたタスクに限定されている。
- 未解決の正確性の指摘がない。

## 報告

最後に次を報告する:

- 変更したファイル
- 実行したテスト
- 残存リスク
- 人間の承認が必要な事項
```

スキルの実行例は下記の通りです。

```text
/tdd-loop issue #123 を実装する
```

Claude Codeではカスタムコマンドがスキルに統合されており、`.claude/commands/deploy.md` と `.claude/skills/deploy/SKILL.md` はどちらも `/deploy` として動作すると説明されています。([Extend Claude with skills](https://code.claude.com/docs/en/skills))

補足: 上の例の `disable-model-invocation: true` は、モデルによる自動起動を禁止し、ユーザーが `/tdd-loop` で手動起動するときだけ使える設定です。このためモデルが自律的に回す `.claude/loop.md` や `/goal` のループからは、このスキルを直接呼び出せません（§概要で「スキル相当の手順を回す」と書いたのはこのためです）。ループに手順を自動で回させたい場合は、このフラグを外してモデルの起動を許すか、フラグは残したまま同じ手順を `loop.md` 側にも書きます。なお §5「実行例」の `/goal` ＋ `/tdd-loop` の併用はこの制約と両立します。直接呼べないのはモデルの自動起動だけで、そこではユーザーが `/tdd-loop` を一度起動して手順を読み込ませ（`/goal` が継続を担う）、継続中はモデルが再起動しなくても手順はコンテキストに残るためです。

スキルは `.claude/skills/<name>/` に置けば自動で認識され、`/<name>` として呼び出せます。Claude Code はスキルディレクトリの変更を監視するため、セッション中に追加・編集・削除したスキルは再起動なしで反映されます（ただしセッション開始時に存在しなかった最上位の `.claude/skills/` ディレクトリを新規作成した場合だけは、監視対象に加えるため再起動が要ります）。([Extend Claude with skills](https://code.claude.com/docs/en/skills))

---

### 4.4 `.claude/loop.md` は「裸の `/loop` の既定プロンプト」にする

`.claude/loop.md` は、裸の `/loop` を打ったときの既定プロンプトを置き換えるファイルです。プロジェクトレベルの `.claude/loop.md` が、ユーザーレベルの `~/.claude/loop.md` より優先されます。これは「複数タスクの一覧」ではなく、裸の `/loop` に対する単一の既定プロンプトです。([Run prompts on a schedule](https://code.claude.com/docs/en/scheduled-tasks))

例：

```md
現在のブランチを確認し、作業を安全に前へ進める。

各イテレーションで:

1. 現在の git ステータスを確認する。
2. 現在のブランチに対するアクティブな PR があるか確認する。
3. CI が失敗していれば、失敗したジョブのログを取得し、根本原因を診断し、最小の安全な修正を行い、関連するチェックを実行する。
4. レビューコメントがあれば、正確性に関するコメントから先に対応する。
5. ローカルテストが失敗していれば、TDD ループに従う:
   - 再現
   - 診断
   - 修正
   - 狭いテストを再実行
   - 広い範囲のチェックを再実行
6. ブランチがグリーンで静かなら、簡潔なステータス更新を1つ返す。

無関係なリファクタを始めない。
現在の会話で明示的に許可されていない限り、プッシュ・ブランチ削除・履歴の書き換え・不可逆な操作を行わない。
```

実行例は下記の通りです。

```text
/loop
```

または固定間隔にするなら：

```text
/loop 10m
```

`.claude/loop.md` はプレーンな Markdown で、構造は必須ではありません。編集は次のイテレーションから反映され、25,000 バイトを超える内容は切り詰められると説明されています。([Run prompts on a schedule](https://code.claude.com/docs/en/scheduled-tasks))

---

### 4.5 `/loop` は「時間ベース」、`/goal` は「条件ベース」と考える

ここはかなり重要です。

`/loop` は、一定間隔または Claude が選ぶ間隔でプロンプトを再実行します。間隔とプロンプトの両方、または片方だけを指定できます。たとえば `/loop 5m デプロイを確認する` は固定間隔、`/loop デプロイを確認する` は Claude がイテレーションごとに間隔を選ぶ動作です。([Run prompts on a schedule](https://code.claude.com/docs/en/scheduled-tasks))

一方、`/goal` は「条件が満たされるまで、ターンごとに継続する」用途です。公式ドキュメントでは、`/goal`、`/loop`、Stop フックの違いとして、`/goal` は前のターンが終わると次が始まり、条件を満たしたと判定されたら止まる、と説明されています。([Keep working toward a goal](https://code.claude.com/docs/en/goal))

ただし重要な限界があります。`/goal` の完了判定は別の高速モデルが行い、それは「Claude が会話に表出した出力テキスト」だけを見て判断します。テストを実際に実行して確認するわけではないため、`/goal` はグリーンを機械的に保証しません（Claude が「通った」と述べれば、その文面を信じて停止しうる）。したがって条件は、出力から検証可能な形（実行したコマンドとその結果を提示させる形）で書く必要があります。グリーンの機械的な担保は、§4.6 の Stop フックや実際のテスト実行に任せます。

TDD では、`/goal` は「グリーンになるまで止めずに回し続ける」継続トリガとして使えます。`/goal` で完了条件を打ってからタスクを指示する具体例は §5「実行例」にまとめています。

`/goal` は条件を満たさない限りターンを跨いで回り続けるため、満たせない条件を与えると延々と継続しトークンを浪費しえます（公式にイテレーション上限の保証はありません）。暴走を避けるには条件に上限を含め（例: 「または20ターンで停止する」）、止めたいときは `/goal clear`（`stop`／`off`／`reset`／`cancel` などのエイリアスも可）を打ちます。([Keep working toward a goal](https://code.claude.com/docs/en/goal))

`/loop` は「CI が終わったか定期的に見る」「PR コメントが増えていないか見る」「デプロイ完了を待つ」用途に向いています。`/goal` は「（出力から判定できる）完了条件まで止めずに進める」用途に向いています。

---

### 4.6 Stop フックで「検証が通るまで止めない」ゲートを作る

より強いワークフローループにしたい場合は、Stop フックを使います。Claude Codeのフックは、Claudeのライフサイクル上の特定ポイントで自動実行されるユーザー定義コマンドです。`CLAUDE.md` のような助言ではなく、必ず実行される決定的な制御として使えます。([Automate actions with hooks](https://code.claude.com/docs/en/hooks-guide))

たとえば、Claude が「完了しました」と言う前に、必ずテスト・lint・型チェックを通す Stop フックを置けます。

`.claude/settings.json`：

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "./scripts/claude-stop-gate.sh"
          }
        ]
      }
    ]
  }
}
```

`scripts/claude-stop-gate.sh`：

```bash
#!/usr/bin/env bash
set -uo pipefail

# フック起因の継続ループを防ぐ: すでに Stop フックのブロックで継続中なら、何もせず停止を許可する。
input=$(cat)
if [ "$(printf '%s' "$input" | jq -r '.stop_hook_active')" = "true" ]; then
  exit 0
fi

failures=()

npm test >/tmp/claude-test.log 2>&1 || failures+=("npm test が失敗: /tmp/claude-test.log")
npm run typecheck >/tmp/claude-typecheck.log 2>&1 || failures+=("npm run typecheck が失敗: /tmp/claude-typecheck.log")
npm run lint >/tmp/claude-lint.log 2>&1 || failures+=("npm run lint が失敗: /tmp/claude-lint.log")

if [ ${#failures[@]} -gt 0 ]; then
  printf '%s\n' "${failures[@]}" >&2
  echo "停止する前に失敗を修正してください。" >&2
  exit 2
fi

exit 0
```

Claude Codeのフックでは、終了コード `2` がブロッキングエラーとして扱われます。`Stop` イベントで終了コード `2` を返すと、Claudeが停止するのを防ぎ、会話を継続させます。([Hooks reference](https://code.claude.com/docs/en/hooks))

このスクリプト冒頭の `stop_hook_active` チェックは省略できません。これがないと、テストを通せないとき Stop フックが停止を妨げ続け、Claude がイールド（停止）できない実質的な無限ループになります。公式仕様では連続8回ブロックすると Stop フックが強制的に解除され（`CLAUDE_CODE_STOP_HOOK_BLOCK_CAP` で上限を変更可）、`stop_hook_active` はその継続中かどうかを判定するためのフラグです。なおこのガードは `jq` に依存する点にも注意してください。また `Stop` フックは matcher を取らず（上の JSON から `matcher` を省いたのはこのためで、付けても無視されます）、Claude が応答を終えようとするたびに毎回発火します。このため実装と無関係な普通の応答でも上記の検証が走り、赤ければ停止できません。スクリプトは事前に実行可能にしておきます（`chmod +x scripts/claude-stop-gate.sh`）。

ただし、これは強い制御です。大規模テストを毎回走らせると重くなるので、最初は「lint だけ」「該当テストだけ」など軽いゲートから始めた方がよいです。

#### Stop フックを有効にする

スクリプトを作っただけでは発火しません。次の手順で有効化します。

1. スクリプトに実行権限を与える（前述の `chmod +x`）。ガードが使う `jq` も導入しておく。
2. 上の `Stop` フック設定を `.claude/settings.json` に登録する。スコープは、チームで共有するなら `.claude/settings.json`（コミット可）、個人用なら `.claude/settings.local.json`（gitignore 対象）、全プロジェクト共通なら `~/.claude/settings.json` で、優先順位は local > project > user、複数スコープのフックは同じイベントで併せて実行されます。([Automate actions with hooks](https://code.claude.com/docs/en/hooks-guide))
3. Claude Code は設定ファイルを監視してライブリロードするため、`settings.json` への `hooks` の追加・編集はセッションを再起動しなくても反映されます。([Settings](https://code.claude.com/docs/en/settings)) 登録できたかは `/hooks` で確認します。これはフックイベントごとに設定数を一覧表示する読み取り専用メニューで、追加・変更は設定ファイルを直接編集します。([Automate actions with hooks](https://code.claude.com/docs/en/hooks-guide))
4. フックの `command` が実行される作業ディレクトリは公式に規定されていないため、`./scripts/...` のような相対パスが解決されない場合は `$CLAUDE_PROJECT_DIR`（プロジェクトルートの絶対パス）でパスを組み立てます。([Automate actions with hooks](https://code.claude.com/docs/en/hooks-guide))
5. スクリプト内の検証コマンド（`npm test` など）を対象プロジェクトのものへ差し替える。
6. わざとテストを失敗させて、Claude が停止しようとしたときにフックがブロックして継続するか、成功状態では普通に停止できるかを確認する。

---

### 4.7 レビューループを入れる

`tdd-loop` スキルにはレビュー手順（差分を新規サブエージェントまたは `/code-review` でレビューし、指摘を修正して再検証する）が既に内蔵されているため、`tdd-loop` を回している場合は本節の指示を別途打つ必要はありません。本節が役立つのは、スキルを使わない実装や、Claude をレビュー専任にする場合（例: Codex が実装し Claude がレビューする運用）の独立レビューです。

自律的に長く回すほど、独立レビューを入れた方が安全です。Claude Code公式のベストプラクティスでも、完了扱いにする前に新しいコンテキストのサブエージェントで差分をレビューさせることが推奨されています。`/code-review` は現在の差分を新規サブエージェントでレビューし、正確性のバグや改善点を返すスキルです。([Best practices for Claude Code](https://code.claude.com/docs/en/best-practices))

`/goal` ループ中でも、モデルは新規サブエージェント（Task）を起動して独立レビューを自律実行できます（§5 の完了条件はこれを前提にしています）。`/code-review` はこの独立レビューをスキル化したもので、`tdd-loop` と違いモデル自動起動を塞いでいないため利用できますが、確実に使えるのは常時起動できるサブエージェント経路です。

実装後に入れる指示例：

```text
現在の差分を、タスクに照らして新規のサブエージェントでレビューする。
正確性・不足しているテスト・エッジケース・スコープクリープに注目する。
完了をブロックすべき指摘だけを報告する。
その後、ブロッキングな指摘があれば修正し、検証を再実行する。
```

または直接：

```text
/code-review high --fix
```

`/code-review` は effort 段階（`low`〜`ultra`）と `--fix`（作業ツリーへ修正を適用）・`--comment`・対象指定を受け付けます。([Commands](https://code.claude.com/docs/en/commands))

大きい変更では、実装エージェントとレビューエージェントを分けるだけで、かなり品質が安定します。

---

## 5. 実行例

```text
/goal 関連テスト・型チェック・lint を実行して結果(exit 0)を提示し、新規サブエージェント（または /code-review）でレビューしてブロッキング指摘がないことを示す。または20ターンで停止する。

/tdd-loop issue #123 を実装する
```

PRやCIの保守を回したいとき：

```text
/loop CI が通ったか確認し、レビューコメントがあれば対応する
```

`.claude/loop.md` を使いたいとき：

```text
/loop
```

---

## 6. `/loop` の限界も把握しておく

`/loop` は現在の CLI セッション内での短時間のポーリング向けです。Claude Codeのスケジュール機能比較では、`/loop` は「現在の CLI セッション」で動き、起動中のセッションが必要で、ローカルファイルにはアクセスできますが、永続的に回す用途では Routines、Desktop scheduled tasks、GitHub Actions が推奨されています。([Run prompts on a schedule](https://code.claude.com/docs/en/scheduled-tasks))

また、セッションスコープのスケジューリングには制約があります。タスクは Claude Code が起動していてアイドルのときだけ発火し、ターミナルを閉じたりセッションが終了したりすると止まります。繰り返しタスクは作成から7日後に自動的に期限切れになります。([Run prompts on a schedule](https://code.claude.com/docs/en/scheduled-tasks))

実行中の `/loop` を手動で止めるには、次のイテレーションを待っている間に Esc を押します。保留中の起動がクリアされ、以降は発火しません。([Run prompts on a schedule](https://code.claude.com/docs/en/scheduled-tasks))

加えて、Amazon Bedrock / Google Vertex AI / Microsoft Foundry 経由で利用している場合は `/loop` の挙動が変わります。`.claude/loop.md` は読まれず、引数なしの bare `/loop` は maintenance prompt を実行せず使用法（usage）表示のみになり、プロンプトを渡して間隔を省いた `/loop` は動的間隔ではなく固定10分間隔で実行されます（2026-06-09 時点）。

なので、使い分けはこうです。

| 目的                    | 使うもの                                                |
| --------------------- | --------------------------------------------------- |
| 今の作業を完了条件（出力で判定）まで進める        | `/goal`                                             |
| 現在セッションでCIやデプロイを見張る   | `/loop`                                             |
| 裸の `/loop` の既定動作を設計する | `.claude/loop.md`                                   |
| 毎回同じ作業手順を再利用する        | スキル                                                |
| 絶対に実行したい検証・禁止ルール      | フック                                                |
| ノートPCを閉じても回したい        | Routines / Desktop scheduled tasks / GitHub Actions |
| 独立レビューを入れたい           | サブエージェント / `/code-review`                     |

用途別には、**Claude Code を実装メインにするなら `/goal + tdd-loop スキル + Stop フック`**、**Codex 実装後のレビュー担当にするなら `code-review スキル + レビュー専用スキル + 必要に応じて /loop で PR 監視`** が一番きれいです。
