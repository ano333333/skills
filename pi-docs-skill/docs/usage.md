# Pi の使い方

> 要約: このページは、Quickstart に載せきれない日常的な Pi の使い方をまとめています。対話 UI、スラッシュコマンド、メッセージキュー、セッション、コンテキストファイル、CLI オプション、設計思想まで一通り確認できます。

このページは、Quickstart ページには収まりきらない日常的な利用詳細をまとめたものです。

## 対話モード

<p align="center"><img src="images/interactive-mode.png" alt="Interactive Mode" width="600"></p>

インターフェースは 4 つの主要領域から構成されます。

- **Startup header** - ショートカット、読み込まれたコンテキストファイル、プロンプトテンプレート、スキル、拡張
- **Messages** - ユーザーメッセージ、アシスタント応答、ツール呼び出し、ツール結果、通知、エラー、拡張 UI
- **Editor** - 入力欄。枠線の色は現在の思考レベルを示します
- **Footer** - 作業ディレクトリ、セッション名、トークン/キャッシュ使用量、コスト、コンテキスト使用量、現在のモデル

エディタは `/settings` のような組み込み UI や、カスタム拡張 UI によって一時的に置き換えられることがあります。

### エディタ機能

| 機能 | 操作方法 |
|---------|-----|
| ファイル参照 | `@` を入力してプロジェクトファイルをあいまい検索 |
| パス補完 | Tab でパス補完 |
| 複数行入力 | Shift+Enter、または Windows Terminal では Ctrl+Enter |
| 画像 | Ctrl+V で貼り付け、Windows では Alt+V、またはターミナルへドラッグ |
| シェルコマンド | `!command` で実行し、出力をモデルへ送信 |
| 非表示シェルコマンド | `!!command` で実行し、出力をモデルへ送らない |
| 外部エディタ | Ctrl+G で `$VISUAL` または `$EDITOR` を開く |

全ショートカットとカスタマイズ方法は [Keybindings](keybindings.md) を参照してください。

## スラッシュコマンド

エディタで `/` を入力するとコマンド補完が開きます。拡張は独自コマンドを登録でき、スキルは `/skill:name` として使え、プロンプトテンプレートは `/templatename` で展開されます。

| コマンド | 説明 |
|---------|-------------|
| `/login`, `/logout` | OAuth または API キー認証情報を管理 |
| `/model` | モデルを切り替える |
| `/scoped-models` | Ctrl+P 循環対象のモデルを有効化/無効化 |
| `/settings` | 思考レベル、テーマ、メッセージ配送、トランスポートを設定 |
| `/resume` | 過去のセッションから選択 |
| `/new` | 新しいセッションを開始 |
| `/name <name>` | セッション表示名を設定 |
| `/session` | セッションファイル、ID、メッセージ数、トークン数、コストを表示 |
| `/tree` | セッション中の任意地点へ移動してそこから継続 |
| `/fork` | 以前のユーザーメッセージから新しいセッションを作成 |
| `/clone` | 現在のアクティブブランチを新しいセッションへ複製 |
| `/compact [prompt]` | 任意の指示付きで手動コンテキスト圧縮 |
| `/copy` | 最後のアシスタントメッセージをクリップボードへコピー |
| `/export [file]` | セッションを HTML としてエクスポート |
| `/share` | 非公開 GitHub gist へアップロードし、共有用 HTML リンクを生成 |
| `/reload` | キーバインド、拡張、スキル、プロンプト、コンテキストファイルを再読込 |
| `/hotkeys` | すべてのキーボードショートカットを表示 |
| `/changelog` | バージョン履歴を表示 |
| `/quit` | Pi を終了 |

## メッセージキュー

エージェントがまだ処理中でも、メッセージを送信できます。

- **Enter** は steering message をキューします。現在のアシスタントターンがツール呼び出しの実行を終えたあとに配送されます。
- **Alt+Enter** は follow-up message をキューします。エージェントがすべての作業を終えたあとに配送されます。
- **Escape** は中断し、キューされたメッセージをエディタへ戻します。
- **Alt+Up** はキューされたメッセージをエディタへ取り戻します。

Windows Terminal では Alt+Enter が既定で全画面表示です。Pi にショートカットを受け取らせたい場合は [Terminal setup](terminal-setup.md) の説明に従って再割り当てしてください。

配送方法は [Settings](settings.md) の `steeringMode` と `followUpMode` で設定します。

## セッション

セッションは `~/.pi/agent/sessions/` に自動保存され、作業ディレクトリごとに整理されます。

```bash
pi -c                  # 直近のセッションを続ける
pi -r                  # セッションを一覧表示して選ぶ
pi --no-session        # エフェメラルモード。保存しない
pi --name "my task"    # 起動時にセッション表示名を設定
pi --session <path|id> # 特定のセッションファイルまたは ID を使う
pi --fork <path|id>    # セッションを新しいセッションファイルへ fork する
```

便利なセッションコマンド:

- `/session` は現在のセッションファイルと ID を表示します。
- `/tree` はファイル内のセッションツリーを移動し、放棄したブランチを要約できます。
- `/fork` は以前のユーザーメッセージから新しいセッションを作成します。
- `/clone` は現在のアクティブブランチを新しいセッションファイルへ複製します。
- `/compact` は古いメッセージを要約してコンテキストを空けます。

詳細は [Sessions](sessions.md) と [Compaction](compaction.md) を参照してください。

## コンテキストファイル

Pi は起動時に、次の場所から `AGENTS.md` または `CLAUDE.md` を読み込みます。

- グローバル指示用の `~/.pi/agent/AGENTS.md`
- 現在の作業ディレクトリから親方向へたどった各ディレクトリ
- 現在のディレクトリ

コンテキストファイルには、プロジェクトの慣習、実行コマンド、安全ルール、好みなどを記述します。読み込みを無効にするには `--no-context-files` または `-nc` を使います。

### システムプロンプトファイル

既定のシステムプロンプトを置き換えるには、次を使います。

- プロジェクト用の `.pi/SYSTEM.md`
- グローバル用の `~/.pi/agent/SYSTEM.md`

既定プロンプトを置き換えずに追記したい場合は、どちらの場所でも `APPEND_SYSTEM.md` を使います。

## セッションのエクスポートと共有

`/export [file]` を使うと、セッションを HTML として書き出せます。

`/share` を使うと、共有用 HTML リンク付きの非公開 GitHub gist へアップロードできます。

Pi をオープンソース作業に使い、モデル・プロンプト・ツール・評価の研究向けにセッションを公開したい場合は [`badlogic/pi-share-hf`](https://github.com/badlogic/pi-share-hf) を参照してください。セッションを Hugging Face datasets に公開します。

## CLI リファレンス

```bash
pi [options] [@files...] [messages...]
```

### Package コマンド

```bash
pi install <source> [-l]     # package をインストール。-l はプロジェクトローカル
pi remove <source> [-l]      # package を削除
pi uninstall <source> [-l]   # remove の別名
pi update [source|self|pi]   # Pi と package を更新。固定 git ref も再調整
pi update --extensions       # package のみ更新。固定 git ref も再調整
pi update --self             # Pi 本体のみ更新
pi update --extension <src>  # 1 つの package を更新
pi list                      # インストール済み package 一覧
pi config                    # package リソースの有効/無効を設定
```

これらのコマンドは Pi CLI 自体ではなく、pi package を管理します。Pi 本体のアンインストールは [Quickstart](quickstart.md#アンインストール) を参照してください。

[Pi Packages](packages.md) には、package の取得元とセキュリティ上の注意があります。

### モード

| フラグ | 説明 |
|------|-------------|
| default | 対話モード |
| `-p`, `--print` | 応答を出力して終了 |
| `--mode json` | すべてのイベントを JSON Lines として出力。[JSON mode](json.md) 参照 |
| `--mode rpc` | stdin/stdout 上の RPC モード。[RPC mode](rpc.md) 参照 |
| `--export <in> [out]` | セッションを HTML へエクスポート |

print モードでは、パイプ経由の stdin も読み取り、初期プロンプトへ統合します。

```bash
cat README.md | pi -p "Summarize this text"
```

### モデルオプション

| オプション | 説明 |
|--------|-------------|
| `--provider <name>` | `anthropic`、`openai`、`google` などのプロバイダ |
| `--model <pattern>` | モデルパターンまたは ID。`provider/id` と任意の `:<thinking>` をサポート |
| `--api-key <key>` | 環境変数を上書きする API キー |
| `--thinking <level>` | `off`、`minimal`、`low`、`medium`、`high`、`xhigh` |
| `--models <patterns>` | Ctrl+P で巡回するモデルのカンマ区切りパターン |
| `--list-models [search]` | 利用可能なモデルを一覧表示 |

### セッションオプション

| オプション | 説明 |
|--------|-------------|
| `-c`, `--continue` | 直近のセッションを続ける |
| `-r`, `--resume` | セッションを一覧表示して選ぶ |
| `--session <path\|id>` | 特定のセッションファイルまたは部分 UUID を使う |
| `--fork <path\|id>` | セッションファイルまたは部分 UUID から新しいセッションを作る |
| `--session-dir <dir>` | セッション保存先ディレクトリを指定 |
| `--no-session` | エフェメラルモード。保存しない |
| `--name <name>`, `-n <name>` | 起動時にセッション表示名を設定 |

### ツールオプション

| オプション | 説明 |
|--------|-------------|
| `--tools <list>`, `-t <list>` | 組み込み・拡張・カスタムツールの許可リストを指定 |
| `--exclude-tools <list>`, `-xt <list>` | 特定の組み込み・拡張・カスタムツールを無効化 |
| `--no-builtin-tools`, `-nbt` | 組み込みツールのみ無効化し、拡張/カスタムツールは残す |
| `--no-tools`, `-nt` | すべてのツールを無効化 |

組み込みツール: `read`、`bash`、`edit`、`write`、`grep`、`find`、`ls`。

### リソースオプション

| オプション | 説明 |
|--------|-------------|
| `-e`, `--extension <source>` | パス、npm、git から拡張を読み込む。複数指定可 |
| `--no-extensions` | 拡張の自動検出を無効化 |
| `--skill <path>` | スキルを読み込む。複数指定可 |
| `--no-skills` | スキルの自動検出を無効化 |
| `--prompt-template <path>` | プロンプトテンプレートを読み込む。複数指定可 |
| `--no-prompt-templates` | プロンプトテンプレートの自動検出を無効化 |
| `--theme <path>` | テーマを読み込む。複数指定可 |
| `--no-themes` | テーマの自動検出を無効化 |
| `--no-context-files`, `-nc` | `AGENTS.md` と `CLAUDE.md` の自動検出を無効化 |

`--no-*` と明示的なフラグを組み合わせると、設定を無視して必要なものだけを読み込めます。例:

```bash
pi --no-extensions -e ./my-extension.ts
```

### その他のオプション

| オプション | 説明 |
|--------|-------------|
| `--system-prompt <text>` | 既定プロンプトを置き換える。コンテキストファイルとスキルは引き続き追記される |
| `--append-system-prompt <text>` | システムプロンプトへ追記 |
| `--verbose` | 起動時の詳細表示を強制 |
| `-h`, `--help` | ヘルプを表示 |
| `-v`, `--version` | バージョンを表示 |

### ファイル引数

メッセージに含めるファイルには `@` を付けます。

```bash
pi @prompt.md "Answer this"
pi -p @screenshot.png "What's in this image?"
pi @code.ts @test.ts "Review these files"
```

### 例

```bash
# 初期プロンプト付きの対話モード
pi "List all .ts files in src/"

# 非対話
pi -p "Summarize this codebase"

# stdin をパイプした非対話
cat README.md | pi -p "Summarize this text"

# 名前付きワンショットセッション
pi --name "release audit" -p "Audit this repository"

# 別モデル
pi --provider openai --model gpt-4o "Help me refactor"

# プロバイダ接頭辞付きモデル
pi --model openai/gpt-4o "Help me refactor"

# 思考レベル省略記法付きモデル
pi --model sonnet:high "Solve this complex problem"

# モデル循環を制限
pi --models "claude-*,gpt-4o"

# 読み取り専用モード
pi --tools read,grep,find,ls -p "Review the code"

# 他は残したまま 1 つの拡張または組み込みツールだけ無効化
pi --exclude-tools ask_question
```

### 環境変数

| 変数 | 説明 |
|----------|-------------|
| `PI_CODING_AGENT_DIR` | 設定ディレクトリを上書き。既定は `~/.pi/agent` |
| `PI_CODING_AGENT_SESSION_DIR` | セッション保存ディレクトリを上書き。`--session-dir` が優先 |
| `PI_PACKAGE_DIR` | package ディレクトリを上書き。Nix/Guix の store path で有用 |
| `PI_OFFLINE` | 起動時のネットワーク処理を無効化。更新確認、package 更新確認、install/update telemetry を含む |
| `PI_SKIP_VERSION_CHECK` | 起動時の Pi バージョン更新確認をスキップ。`pi.dev` への最新バージョン問い合わせを抑止 |
| `PI_TELEMETRY` | install/update telemetry を上書き: `1`/`true`/`yes` または `0`/`false`/`no`。更新確認自体は無効化しない |
| `PI_CACHE_RETENTION` | 対応環境で拡張 prompt cache を使うには `long` を指定 |
| `VISUAL`, `EDITOR` | Ctrl+G 用の外部エディタ |

## 設計原則

Pi はコアを小さく保ち、ワークフロー固有の振る舞いは拡張、スキル、プロンプトテンプレート、package 側へ押し出します。

そのため、組み込み MCP、sub-agent、権限ポップアップ、plan mode、to-do、バックグラウンド bash は意図的に含まれていません。そうしたワークフローは拡張や package として自作・導入するか、コンテナや tmux のような外部ツールを使って構築できます。

背景にある考え方の詳細は [blog post](https://mariozechner.at/posts/2025-11-30-pi-coding-agent/) を参照してください。
