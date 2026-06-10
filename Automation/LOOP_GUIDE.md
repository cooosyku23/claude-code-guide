# Claude Code `.claude/loop.md` 記述ガイド

`Automation/GUIDE.md` §4 で触れた `.claude/loop.md` の書き方を、独立した実務ガイドとして掘り下げたものです。`loop.md` は、引数なしの `/loop`（bare `/loop`）が実行する組み込みの maintenance prompt を、自分の指示で置き換えるためのファイルです。2026年6月8日時点の Anthropic 公式ドキュメントを照合して記述します。挙動・制限は変更される可能性があるため、時間に依存する事実には照合日を併記します。

主な一次情報:

- [Run prompts on a schedule](https://code.claude.com/docs/en/scheduled-tasks)（`loop.md` の正式仕様。2026-06-08 照合）
- [Keep working toward a goal](https://code.claude.com/docs/en/goal)
- [Slash commands reference](https://code.claude.com/docs/en/commands)
- [How Claude remembers your project](https://code.claude.com/docs/en/memory)

注: 本ガイドの前提として、Scheduled tasks（`/loop` を含む）は Claude Code v2.1.72 以降で利用できます（2026-06-08 時点）。

---

## 1. `loop.md` とは何か

`loop.md` は、bare `/loop` の既定プロンプトを置き換える1つのファイルです。`/loop` は渡す引数によって挙動が変わり、`loop.md` が効くのは「プロンプトを渡さない」場合だけです。

| 渡すもの | 例 | 挙動 |
| --- | --- | --- |
| 間隔とプロンプト | `/loop 5m デプロイを確認` | プロンプトを固定間隔（cron）で実行する |
| プロンプトのみ | `/loop デプロイを確認` | プロンプトを、Claude が毎回動的に選ぶ間隔で実行する |
| 間隔のみ、または何も渡さない | `/loop 15m`・`/loop` | 組み込みの maintenance prompt を実行する。`loop.md` があればそれを実行する |

ここから読み取るべき要点は2つです。

第一に、`loop.md` は **bare `/loop` 用の単一の既定プロンプト** を定義するもので、複数タスクの一覧ではありません。複数のスケジュールを並べて回したい場合は `/loop <prompt>` を個別に使うか、Claude へ自然言語で直接スケジュールを依頼します。

第二に、コマンドラインでプロンプトを渡したときは `loop.md` は無視されます。`/loop デプロイを確認` のようにプロンプトを与えた `/loop` は、`loop.md` の内容を読みません。

---

## 2. 配置場所と優先順位

`loop.md` は次の2か所が探索され、最初に見つかった方が使われます。

| パス | スコープ |
| --- | --- |
| `.claude/loop.md` | プロジェクト単位。両方ある場合はこちらが優先される |
| `~/.claude/loop.md` | ユーザー単位。独自の `loop.md` を持たないプロジェクトに適用される |

どちらの場所にも `loop.md` がなければ、bare `/loop` は組み込みの maintenance prompt にフォールバックします。

---

## 3. 書き方のルール（核心）

ここが `loop.md` の最重要ポイントです。公式仕様では、`loop.md` は **構造の決まりがないプレーンな Markdown** であり、「`/loop` のプロンプトを直接打つつもりで書く」のが基本です。

このため、スキル（`SKILL.md`）やカスタムコマンドと違い、`name:` / `description:` のような **YAML frontmatter は不要** です。`loop.md` は `/loop` のプロンプトとしてそのまま使われるので、プロンプト本文だけを書きます。公式は frontmatter を要求せず、付けた場合にどう扱われるかも示していないため、`SKILL.md` の癖で frontmatter を付けないよう注意してください。

そのほかの仕様は次のとおりです。

- 簡潔に保つこと。25,000バイトを超えた分は切り捨てられます（2026-06-08 時点）。これは文字数ではなくバイト数で、UTF-8 では日本語1文字が約3バイトになるため、日本語で書く場合の目安は約8,000字です。
- `loop.md` の編集は次のイテレーションから反映されます。ループを回しながら指示を調整できます（フォールバック挙動は §2 のとおりで、どちらの場所にも `loop.md` がなければ組み込みの maintenance prompt が使われます）。

`SKILL.md`（frontmatter 必須）と `loop.md`（frontmatter 不要）の違いを取り違えやすいので、ここだけは明確に区別してください。

| ファイル | frontmatter | 中身 |
| --- | --- | --- |
| `.claude/skills/<name>/SKILL.md` | `name:` / `description:` が必須 | 手順本体 |
| `.claude/loop.md` | 不要（付けない） | `/loop` にそのまま打つプロンプト本文 |

---

## 4. bare `/loop` の動作と組み込み maintenance prompt

`loop.md` の中身を設計する前に、置き換える対象である組み込み maintenance prompt の設計を理解しておくと、良いひな型になります。

bare `/loop`（間隔なし）は、毎イテレーション後に Claude が1分〜1時間の範囲で状況に応じた間隔を動的に選びます（ビルド進行中や PR がアクティブなら短く、静かなら長く）。固定間隔にしたい場合は `/loop 15m` のように間隔を付けます。

組み込みの maintenance prompt は、各イテレーションで次の順に作業します。

1. 会話の未完了作業があれば継続する。
2. 現在のブランチの PR の世話をする（レビューコメント・失敗した CI・マージ衝突）。
3. ほかに保留がなければ、bug hunt や simplification などのクリーンアップを回す。

そのうえで、この範囲外の新規の取り組みは始めず、プッシュや削除などの不可逆操作は、トランスクリプトが既に許可した作業の継続時のみ行います。この「範囲を固定し、不可逆操作を限定する」設計を自作 `loop.md` にどう活かすかは §6 で扱います。

---

## 5. 具体例

公式ドキュメントの例は、release ブランチを健全に保つものです（公式の例のため、原文の英語のまま引用します）。

```markdown title=".claude/loop.md"
Check the `release/next` PR. If CI is red, pull the failing job log,
diagnose, and push a minimal fix. If new review comments have arrived,
address each one and resolve the thread. If everything is green and
quiet, say so in one line.
```

`claude/` prefix ブランチ・TDD・draft PR を前提とする運用に寄せると、bare `/loop` の既定としては「現在のブランチの PR を健全な状態へ持っていく」系が自然です。`loop.md` の中身は日本語で書いてもよいため、ここでは日本語で書いた例を示します（ブランチ名・コマンド・GitHub の操作名など英語が自然な語は英語のまま残しています）。

```markdown title=".claude/loop.md"
現在のブランチの PR を見て、健全な状態に持っていってください。

1. CI が失敗しているなら、失敗したジョブのログを読み、最小の根本原因を特定して、
   このブランチに最小限の修正をプッシュしてください。無関係なコードには触れないこと。
2. 未解決のレビューコメントがあれば、それぞれに対応してスレッドを resolve してください。
3. ブランチが base との間でマージ衝突を起こしているなら、保守的に解消し、
   意図が曖昧な箇所では base ブランチ側を優先してください。

制約:
- 現在の `claude/` ブランチから出ないこと。main・develop・release ブランチへプッシュせず、
  PR をマージしないこと。
- 挙動を保つ変更か、テストで裏づけられた変更だけを行うこと。CLAUDE.md のテストコマンドを
  実行し、グリーンな実行を合格ラインとすること。
- 人間の判断を要するものがあれば、推測せず、停止して報告すること。

すべてがグリーンで、やることが残っていなければ、その旨を1行で述べてください。
```

上の2例はどちらも現在ブランチの PR を世話する用途ですが、`loop.md` には bare `/loop` に毎イテレーションさせたい定型作業であれば何でも書けます（PR の世話に限りません。デプロイ監視やドキュメント鮮度チェックなどでも使えます）。

`loop.md` の中身は英語でも日本語でもかまいませんが、`/loop` のプロンプトとしてそのまま読まれるため、`CLAUDE.md` の指示（テストコマンド・ブランチ規約など）と整合させておくと挙動が安定します。

---

## 6. 書くときのコツ

組み込み maintenance prompt の設計（範囲を固定し、不可逆操作を限定し、静かなときは短く終える）から、自作 `loop.md` に効く指針が3つ引けます。

第一に、範囲を狭く固定することです。bare `/loop` は放置されがちなので、「新しい機能を始めない」「このブランチから出ない」と明示しておくと、想定外の変更に広がりません。

第二に、プッシュ・マージ・削除などの不可逆操作を明示的に制限することです。組み込みプロンプトが不可逆操作を「既に許可された作業の継続」に限っているのと同じ姿勢を、自作プロンプトにも書き込みます。

第三に、静観条件を1行で言わせることです。毎イテレーションで長文を吐かせず「すべてグリーン、やることなし」の1行で終わらせると、回しっぱなしでも邪魔になりません。

終了条件の設計をさらに厳密にしたい場合は、時間間隔で繰り返す `/loop` ではなく、検証可能な完了条件で終わる `/goal` も検討します（詳細は `Automation/GUIDE.md` §4「`/goal` で検証可能な終了条件まで継続する」）。

---

## 7. ハマりどころ

`loop.md` を使ううえで、公式ドキュメントが明記している注意点を挙げます。

### Bedrock / Vertex AI / Microsoft Foundry 経由

Amazon Bedrock / Google Vertex AI / Microsoft Foundry 経由で利用している場合、`loop.md` は読まれず、`loop.md` による既定プロンプトのカスタマイズは効きません（2026-06-08 時点）。`/loop` の挙動は渡す引数で2つに分かれます。プロンプトを渡さない bare `/loop` は、maintenance prompt を実行せず使用法（usage）メッセージを表示します。プロンプトを渡して間隔だけを省略した `/loop`（例 `/loop デプロイを確認`）は、Claude が選ぶ動的間隔ではなく固定10分間隔で実行されます。

### session-scoped と7日期限

`/loop` のタスクは session-scoped で、現在の会話に紐づき、新しい会話を始めると消えます。`--resume` / `--continue` では、期限切れでないタスクだけが復元されます。固定間隔の recurring task は作成から7日で期限切れになり、最後に1回実行してから自己削除されます。`loop.md` で定義した既定プロンプトを恒久的に走らせたい用途には向かず、その場合は Routines や Desktop scheduled tasks を使います（`Automation/GUIDE.md` §1・§2 を参照）。

### 停止方法

`/loop` が次のイテレーションを待っている間に停止するには `Esc` を押します。これで保留中の wakeup がクリアされ、ループは再発火しません。動的間隔（self-paced）モードでは、タスクが完了したと判断できた時点で Claude が次の wakeup をスケジュールせず、自分でループを終えることもあります。

---

## 8. `Automation/GUIDE.md` との関係

本ガイドは、`Automation/GUIDE.md` §4「`/loop`・Skills・`/goal` でローカル自律ループを作る」のうち、`.claude/loop.md` の記述方法だけを抜き出して深掘りした補助ガイドです。`/loop` 全体の位置づけ（Routines / Desktop scheduled tasks との使い分け）、`/goal` による終了条件の強制、Skills による反復手順のパッケージ化、実務運用の罠は `GUIDE.md` 本体を参照してください。ワークフローループ全体（`CLAUDE.md`・Skills・`/loop`・`/goal`・Stop フック・レビュー）の設計の見取り図は、もう1本の補助ガイド `Automation/WORKFLOW_LOOP_GUIDE.md` を参照してください。
