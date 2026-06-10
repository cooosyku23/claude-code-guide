# Claude Code Routines と `/loop` 実務ガイド

Claude Code の **Routines**、**Desktop scheduled tasks**、**`/loop`** を実務で使い分けるためのガイドです。2026年5月20日時点の Anthropic 公式ドキュメントを優先し、Routines は research preview である前提で記述します。挙動、制限、API surface は変更される可能性があります。

主な一次情報:

- [Automate work with routines](https://code.claude.com/docs/en/routines)
- [Claude Code on the web](https://code.claude.com/docs/en/claude-code-on-the-web)
- [Run prompts on a schedule](https://code.claude.com/docs/en/scheduled-tasks)
- [Keep working toward a goal](https://code.claude.com/docs/en/goal)
- [Extend Claude with skills](https://code.claude.com/docs/en/skills)
- [Slash commands reference](https://code.claude.com/docs/en/commands)
- [How Claude remembers your project](https://code.claude.com/docs/en/memory)
- [Configure permissions](https://code.claude.com/docs/en/permissions)
- [Introducing routines in Claude Code](https://claude.com/blog/introducing-routines-in-claude-code)

---

## 1. まず使い分けを決める

Claude Code には、自動実行の方式が複数あります。最初に「どこで動かすべきか」を決めると設計が安定します。

| 方式 | 実行場所 | セッションの要否 | ローカルファイルへのアクセス | 最小間隔 | 向いている用途 |
| --- | --- | --- | --- | --- | --- |
| Routines | Anthropic 管理のクラウド | 不要 | なし。GitHub リポジトリは毎回 fresh clone | 1時間 | 定期レビュー、PR監査、アラート対応、ドキュメント鮮度チェック、トリガー連動の実装（修正PR・設計書からのゼロ実装を含む） |
| Desktop scheduled tasks | 自分のマシン | 不要 | あり | 1分 | ローカルDB、ブラウザ、社内VPN、ローカルファイルが必要な処理 |
| `/loop` | 自分のマシンの開いている Claude Code セッション | 必要 | あり | 1分 | デプロイ待ち、CI待ち、短時間のポーリング、自律デバッグ |

判断基準は単純です。

- **PCを閉じても走らせたい**: Routines
- **ローカル環境が必要**: Desktop scheduled tasks
- **今開いている作業セッション内で短期的に回したい**: `/loop`
- **CI/CD に組み込むべきリポジトリネイティブな処理**: GitHub Actions も候補
- **設計書からの実装まで自動化したい**: Routines（トリガー連動の実装。詳細は §2「Routines で実装タスクを動かす」）

Routines はレビューや監査だけの仕組みではなく、バグ修正・機能追加・設計書からのゼロ実装といったコード変更も自動化できます。

---

## 2. Routines の現在の仕様

Routines は、プロンプト、1つ以上の GitHub リポジトリ、Cloud environment、Connectors、Triggers を保存した Claude Code のクラウド実行設定です。クラウド上の Claude Code セッションとして自律実行されるため、実行中に権限確認プロンプトは出ません。

作成・管理は以下から行えます。

- Web: `https://claude.ai/code/routines`
- Desktop app: Routines から New routine を選び、Remote を選択
- CLI: `/schedule`

Routines は Pro、Max、Team、Enterprise プランで Claude Code on the web が有効な場合に利用できます。Team / Enterprise では管理者が organization policy で無効化している場合があります。

Routine は個人の claude.ai account に属します。routine が GitHub や connector 経由で行うコミット、PR、Slack 投稿、Linear ticket 作成などは、接続済みの自分の account による操作として扱われます。

注意点として、CLI の `/schedule` は scheduled routine の作成が中心です。API trigger や GitHub trigger を追加する場合は、現時点では Web UI で編集します。

### Schedule trigger

Schedule trigger は、hourly、daily、weekdays、weekly などのプリセット、または将来の1回限りの実行に使います。

カスタム cron を使いたい場合は、近いプリセットで作成したあとに CLI の `/schedule update` で調整します。ただし、Routines の最小間隔は1時間です。1分単位で回す必要があるなら、Routines ではなく Desktop scheduled tasks か `/loop` を選びます。

実務例:

```text
毎週月曜の 9:00 に、直近7日間で main にマージされた PR を走査する。
public API の変更を洗い出し、ドキュメントリポジトリが旧挙動に言及していないか確認し、
最小限のドキュメント更新を行う draft PR を作成する。更新が不要なら、設定済みの
Slack connector に短い要約を投稿する。
```

### API trigger

API trigger は、routine ごとの HTTP endpoint に Bearer token 付きで `POST` すると、新しい Claude Code セッションを開始します。Sentry、Datadog、CD pipeline、社内ダッシュボードなどから呼び出す用途に向いています。

リクエストボディには任意の `text` フィールドを渡せます。この値は routine の保存済みプロンプトに追加される自由テキストであり、JSON payload として構造解釈されるわけではありません。構造化データを渡す場合も、Claude に「この text は JSON 文字列として読め」とプロンプトで明示します。

例:

```bash
curl -X POST "$CLAUDE_ROUTINE_FIRE_URL" \
  -H "Authorization: Bearer $CLAUDE_ROUTINE_TOKEN" \
  -H "anthropic-version: 2023-06-01" \
  -H "anthropic-beta: experimental-cc-routine-2026-04-01" \
  -H "Content-Type: application/json" \
  -d '{"text":"Sentry アラート SEN-4521 が本番で発火した。このアラート ID を PR の説明に含めること。"}'
```

実務例:

```text
text payload からアラートのコンテキストを読み取る。このリポジトリで該当しそうな担当サービスを特定し、
default branch の最近のコミットを調べ、最小で安全な修正を提案する。claude/ ブランチに draft PR を作成する。
インシデントが解決したとは主張しないこと。triage の要約、変更したファイル、実行したテスト、
マージ前に on-call の人間が確認すべき事項を含めること。
```

### GitHub trigger

GitHub trigger は、接続済みリポジトリの GitHub event に反応して routine を開始します。現時点の公式ドキュメントでは、主な event category は **Pull request** と **Release** です。PR では opened、closed、assigned、labeled、synchronized などの action に反応できます。

PR trigger では、author、title、body、base branch、head branch、labels、draft 状態、merged 状態などで filter できます。各 matching event は独立した新しいセッションを開始します。PR 更新ごとに同じセッションが継続利用される前提で設計しないでください。

実務例:

```text
main を対象とする PR が opened または synchronized され、head branch や変更ファイルから
認証関連の変更が疑われる場合、authentication と authorization への影響だけをレビューする。
具体的なセキュリティ問題には inline コメントを残す。ブロッカーがなければ、要約コメントを1件付ける。
approve もマージもしないこと。
```

### Routines で実装タスクを動かす

Routines はレビューや監査の専用ツールではありません。Routine の実行はフルの Claude Code クラウドセッションであり、シェルコマンドの実行、ファイル編集、`claude/` prefix ブランチへのコミット、draft PR 作成ができます。バグ修正や機能追加に加えて、設計書を渡したゼロからの実装も実行できます。公式ドキュメントの example use cases にも、アラート起因の修正 PR を出す用例や、ある SDK の変更を別言語の SDK へ移植（=再実装）する用例が含まれます。

実装エンジンとしての能力に「アラート起因かゼロ実装か」の差はありません。実務で差が出るのは次の3点です。

**1. 設計書をどこに置くか**

Routine はローカルファイルを見られず、実行ごとにリポジトリを fresh clone します。手元の設計書をそのまま渡せないため、Routine から到達できる場所に置きます。

| 設計書の置き場所 | 渡し方 |
| --- | --- |
| リポジトリにコミット（例 `docs/specs/feature-x.md`） | プロンプトでパスを指定して読ませる。最も確実 |
| Routine の保存プロンプト本体 | 設計書をプロンプトに直書きする |
| API trigger の `text` フィールド | freeform text として保存済みプロンプトに追記される（JSON 構造としては解釈されない） |
| Connector 経由（Google Drive、Linear など） | 該当 connector を routine に含めて読ませる |

**2. 実行中に質問できない**

Routine は完全自律実行で、実行中に確認も質問も出しません。設計書が曖昧でも、Claude は人間に聞き返さずその場の推測で進めます。受け入れ条件、API 仕様、データモデルまで、追加質問なしで実行できるレベルに具体化しておきます。プロンプトには検証可能な成功条件（テストが exit 0、ビルドがグリーンなど）も明記します。

**3. 1セッション = 1 PR で、セッション継続はない**

トリガー 1回ごとに独立した新しいセッションが立ち、セッションをまたいで作業は継続しません。「実装 → レビュー → 同じセッションで作り込み」というループは Routine 単体ではできません。大規模な一括実装は 1 PR が巨大化してレビュー困難になりやすいため、機能単位に分割して投げます。

実務例（設計書からのゼロ実装）:

```text
docs/specs/feature-x.md の設計書を読む。新しい claude/ ブランチでそれを実装する。
その文書を single source of truth として扱う。曖昧な場合は、最小で妥当な解釈を選び、
その種の仮定をすべて PR の説明に列挙する。
プロジェクトのテストコマンドとビルドコマンドを実行する。両方が通ることを成功とする。
変更したファイル、実行したテスト、置いた仮定、未解決の疑問を添えて draft PR を作成する。
マージしないこと。main や develop にプッシュしないこと。
```

**Routine と Claude Code on the web の使い分け**

実装エンジン自体は、Routines も Claude Code on the web も同じクラウドセッションです。Routines はそこに「スケジュール／トリガーで起動する」レイヤーを足したものなので、実装タスクを振るときは「トリガーが要るか」で選びます。

- **同じ実装フローを繰り返したい**（例: `spec-ready` ラベルが付いた PR を実装する、毎週たまった spec をまとめて実装する）: Routine の GitHub trigger / Schedule trigger を使う。
- **一回きりの実装を、途中で対話しながら詰めたい**: Routine よりも Claude Code on the web のクラウドセッションを直接起動するほうが自然です。トリガーで包む意味が薄く、実行中に方針を相談できます。
- **一回きりだが完全自律でよい**: Routine を作成し、`Run now` で即実行するか、one-off schedule で起動します。

---

## 3. Routines の権限、環境、Connectors

Routines はクラウドで自律実行されるため、最初の設定が安全性を大きく左右します。

### Branch permissions

Routine は実行ごとにリポジトリを clone し、明示しない限り default branch から始めます。デフォルトでは、Claude がプッシュできるブランチは `claude/` prefix のブランチに制限されます。`main` や `develop` などの長期運用ブランチに直接プッシュさせたい場合だけ、リポジトリごとに Allow unrestricted branch pushes を有効化します。

実務では、原則として以下をプロンプトに入れます。

```text
claude/ を接頭辞とする新しいブランチを作成する。draft PR を作成する。main・develop・
release branch にはプッシュしないこと。PR をマージしないこと。
```

### Connectors

Routines は claude.ai account に接続された MCP connectors を使えます。Slack、Linear、Google Drive などに書き込む connector を routine に含めると、実行中に確認なしでその connector のツールを使えます。

重要な運用ルール:

- routine 作成時は接続済み connectors が既定で含まれるため、不要な connector は外す。
- Slack 通知だけでよい routine に、Linear や Google Drive まで渡さない。
- ローカルで `claude mcp add` した MCP server は、そのままではクラウド routine の connector list に出ない。
- クラウド routine で使いたい MCP は、claude.ai の connector として追加するか、リポジトリに `.mcp.json` としてコミットする。

### Cloud environment と network access

Routine は Cloud environment の network policy、environment variables、setup script に従って動きます。Network access には `None` / `Trusted` / `Full` / `Custom` の4レベルがあり、Default environment は `Trusted` です。`Trusted` では一般的な package registry や cloud provider API などは許可されますが、任意の外部 host にはアクセスできません（ブロックされた通信は `403` と `x-deny-reason: host_not_allowed` を返します）。

社内 API や独自 domain にアクセスする routine では、Network access を `Custom` に切り替えると現れる `Allowed domains` フィールドに対象 domain を登録します。逆に、外部通信が不要な routine では `Full`（無制限）を避け、`None` または `Trusted` に絞り、connector も必要最小限にします。

---

## 4. `/loop`・Skills・`/goal` でローカル自律ループを作る

短時間のポーリングや、テストが通るまでの反復作業には `/loop` を使います。`/loop` は bundled スキルであり、セッションが開いている間にプロンプトを繰り返し実行します。

例:

```text
/loop 5m デプロイが完了したか確認し、現在の状況を1行で要約する
```

プロンプトだけを渡すと、Claude が各イテレーションの間隔を選ぶ場合があります。

```text
/loop デプロイを確認し、次にいつ確認するかを判断する
```

bare `/loop` は組み込みの maintenance prompt を実行します。project-level の既定プロンプトを定義したい場合は `.claude/loop.md`、user-level なら `~/.claude/loop.md` を使います。`.claude/loop.md` は bare `/loop` の既定プロンプトを置き換えるもので、複数タスクの一覧ではありません。

`.claude/loop.md` の詳しい書き方（配置場所と優先順位・frontmatter 不要のプレーン Markdown・具体例・ハマりどころ）は、補助ガイド `Automation/LOOP_GUIDE.md` にまとめています。ワークフローループ全体の設計（`CLAUDE.md`・Skills・`/loop`・`/goal`・Stop フック・レビューの組み合わせ）の見取り図は、補助ガイド `Automation/WORKFLOW_LOOP_GUIDE.md` にまとめています。

注: Amazon Bedrock / Google Vertex AI / Microsoft Foundry 経由で利用している場合、`/loop` の挙動が異なります。interval を省略した `/loop` は動的間隔ではなく固定10分間隔で実行され、引数なしの bare `/loop` は maintenance prompt を実行せず使用法表示のみになります。

`/loop` の制約:

- Claude Code が起動していて、セッションが開いている必要がある。
- recurring task は作成から7日で期限切れになる。
- セッションを新規開始すると session-scoped task は消える。
- `--resume` や `--continue` では、期限内の task だけ復元される。ただし background Bash や Monitor で動くタスクは、期限内でも復元されない。

### `/goal` で検証可能な終了条件まで継続する

`/goal` は、完了条件が満たされるまで Claude をターン横断で継続させる正式な組み込みコマンドです（公式ドキュメント `code.claude.com/docs/en/goal`）。時間間隔で繰り返す `/loop` と異なり、`/goal` は「条件の達成」で終了します。

完了判定は別の高速モデルが行い、Claude が会話に表出した内容だけを見て条件成立を判断します。ツールを実行して検証するわけではありません。そのため条件は、Claude の出力から検証可能な形で書く必要があります。

- 機能する条件の例: 「`npm test` が exit 0 になり、その結果を提示する」
- 機能しない条件の例: 「コード品質が高い」（出力からは判定できない）

ループの暴走を防ぐには終了条件の設計が要であり、`/goal` はそれをコマンドとして強制する仕組みです。

### Skills で反復手順をパッケージ化する

以前の custom slash commands は現在も `.claude/commands/*.md` として動作しますが、現在の推奨は Skills です。Skills は `.claude/skills/<skill-name>/SKILL.md` に定義します。supporting files、scripts、examples、dynamic context injection などを持てるため、複雑な手順に向いています。

例: `.claude/skills/smart-fix/SKILL.md`

```markdown
---
name: smart-fix
description: 試行回数を区切ったデバッグループで失敗テストを修正する
---

CLAUDE.md のプロジェクトテストコマンドを実行する。
テストが失敗したら:
1. 失敗の出力を読む。
2. 最も小さい根本原因の候補を特定する。
3. その原因に必要なファイルだけを編集する。
4. まず該当する focused test を再実行し、次に全体のテストコマンドを実行する。

修正は5回試したら止める。それでもテストが失敗するなら、試行を続けないこと。
次の内容を含む短い失敗レポートを書く:
- 失敗したコマンド
- 最新のエラー
- 変更したファイル
- 次に人間が取るべき推奨アクション
```

実行:

```text
/smart-fix
```

`/loop` と組み合わせる場合:

```text
/loop 20m /smart-fix
```

ただし、テスト修正のように状態が大きく変わる作業は、無期限に回さず、必ず試行回数・終了条件・人間への引き継ぎ条件をスキル側に書きます。

---

## 5. CLAUDE.md、rules、auto memory の役割

`CLAUDE.md` は Claude Code に読ませる永続的な project instruction です。強制設定ではなく、コンテキストとして読み込まれる指示なので、短く具体的に書くほど守られやすくなります。

置き場所の代表例:

| Scope | 場所 | 用途 |
| --- | --- | --- |
| User | `~/.claude/CLAUDE.md` | 個人の全プロジェクト共通設定 |
| Project | `./CLAUDE.md` または `./.claude/CLAUDE.md` | team で共有する project instruction |
| Local | `CLAUDE.local.md` | コミットしない個人用の補足 |

例:

```markdown
# プロジェクトルール

## ビルドとテスト
- インストール: `pnpm install`
- 型チェック: `pnpm typecheck`
- ユニットテスト: `pnpm test`
- Lint: `pnpm lint`

## エンジニアリングルール
- 挙動を保つ最小の変更を優先する。
- 生成ファイルを直接変更しない。
- データベーススキーマの変更では、同じ PR で migration とテストを更新する。

## PR のルール
- ブランチは `claude/` 接頭辞で作成する。
- 自動化された変更には draft PR を作成する。
- 実行したテストと既知のリスクを PR の説明に含める。
```

大規模リポジトリでは、常に全部を `CLAUDE.md` に詰め込まないでください。

- 常時必要な基本ルール: `CLAUDE.md`
- path や領域ごとのルール: `.claude/rules/*.md`
- 反復可能な手順: `.claude/skills/<name>/SKILL.md`
- Claude が学んだビルドコマンドや debugging insight: auto memory

---

## 6. 実務運用の罠と回避策

### 罠1: Routines のグリーンステータスを「タスク成功」と誤解する

Routine の run list でグリーンステータスになっていても、それは infrastructure error なしでセッションが開始・終了したという意味です。プロンプトの目的が達成されたとは限りません。

対策:

- run トランスクリプトを確認する。
- プロンプトに成功条件を明記する。
- 成功・失敗・未完了を Slack や PR コメントに明示させる。

例:

```text
最後に、SUCCESS / NO_ACTION_NEEDED / NEEDS_HUMAN_REVIEW のいずれかを書く。
理由と、作成した PR やコメントへのリンクを含める。
```

### 罠2: 終了条件が曖昧でループが膨らむ

「いい感じに直して」「失敗しなくなるまで」だけでは、修正と再試行を繰り返して usage を消費しやすくなります。

対策:

- 最大試行回数を指定する。
- 変更してよい範囲を指定する。
- 失敗時の引き継ぎ report を指定する。
- 大きい作業には `/goal` も検討する（詳細は「`/goal` で検証可能な終了条件まで継続する」の項を参照）。

### 罠3: Connector と network access を広げすぎる

Routines は自律実行中に permission プロンプトを出しません。不要な connector や `Full` レベルの network access を渡すと、誤操作や情報露出のリスクが広がります。

対策:

- routine ごとに connector を最小化する。
- Cloud environment を用途別に分ける。
- Allowed domains を必要な domain に絞る。
- 書き込み系 connector を含める場合は、プロンプトに「何を書いてよいか」を具体化する。

### 罠4: クラウド routine がローカル環境を見られると思い込む

Routines はクラウドでリポジトリを clone して実行されます。あなたのローカルファイル、ローカルDB、ローカルに追加した MCP server、開いているブラウザ状態にはアクセスできません。

対策:

- ローカル依存がある処理は Desktop scheduled tasks か `/loop` にする。
- クラウドで必要な secrets は Cloud environment の env vars に設定する。ただし専用の secrets ストアは未提供で、env vars と setup script は環境を編集できる人に可視になるため、環境を用途別に分け編集権限を絞る。
- クラウドで必要なツールのインストールは setup script に寄せる。
- MCP は connector またはコミット済み `.mcp.json` として用意する。

### 罠5: Branch protection だけに頼る

公式仕様では、Routines はデフォルトで `claude/` prefix ブランチにしかプッシュできません。ただし、リポジトリ設定で unrestricted branch pushes を有効にすると、この制限を外せます。また、GitHub 側の branch protection 設計が弱いと、人間の誤マージや CI 不備で壊れた変更が入り得ます。

対策:

- `main`、`develop`、release branch は GitHub branch protection を設定する。
- required checks と code owners を設定する。
- Routine プロンプトには「draft PR を作る」「マージしない」を明記する。
- unrestricted branch pushes は例外扱いにする。

### 罠6: usage limit を固定値として設計する

Routines は通常の subscription usage を消費し、さらに daily routine run cap があります。プランや組織設定、usage credits の有無で実際の上限は変わります。

対策:

- 固定値を運用ドキュメントに書かず、`claude.ai/code/routines` と `claude.ai/settings/usage` を確認する。
- 高頻度の監視は Routines ではなく、外部監視から API trigger で必要時だけ起動する。
- 1時間未満の定期 polling は `/loop`、Desktop scheduled tasks、GitHub Actions などに逃がす。

---

## 7. 推奨テンプレート

Routine や Skill のプロンプトは、以下の形にすると事故が減ります。

```text
目的:
- どんな成果を出すべきか。

範囲:
- Claude が調査・編集してよいリポジトリ・ディレクトリ・ブランチ・ファイル・サービス。

制約:
- 保護されたブランチにプッシュしない。
- PR をマージしない。
- 列挙した connector だけを使う。
- N 回試したら止める。

検証:
- 実行するコマンド。
- 何をもって成功とするか。

出力:
- PR、Slack メッセージ、Linear チケット、または no-op の要約。
- 変更したファイル、実行したテスト、リスク、人間のレビュー手順を含める。

失敗時の対応:
- 行き詰まったら、止めてブロッカーを報告する。
- 無関係な変更で再試行を続けない。
```

Routines と `/loop` は「放っておける魔法」ではなく、事前に境界を決めた agentic workflow として扱うと安定します。特にクラウド実行では、リポジトリ、ブランチ、connectors、network、成功条件を routine ごとに狭く設計してください。
