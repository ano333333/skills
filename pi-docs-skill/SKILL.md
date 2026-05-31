---
name: pi-docs-ja
description: Pi ライブラリの公式ドキュメント一式を日本語で参照するための索引スキル。導入、日常利用、プロバイダ設定、拡張、SDK、RPC、セッション形式、各種プラットフォーム設定を調べたいときに使う。
---

# Pi Docs 日本語索引

Pi の公式 docs (`https://pi.dev/docs/latest`) をベースにした日本語訳版の索引です。各ページは `docs/` 配下に個別 Markdown として保存してあり、この `SKILL.md` は目次と要約一覧として使います。

## 使い方

- Pi の仕様、設定、拡張方法、SDK、RPC 連携、セッション形式を調べたいときは、該当ページを直接開いて参照してください。
- ページは原文のファイル名を維持しているため、公式 docs の内部リンクに追従しやすい構成です。
- 各ページ冒頭の `> 要約:` を先に読むと、必要な資料かどうかをすばやく判断できます。

## Start Here

- [index.md](docs/index.md): Pi 全体の公式ドキュメント目次。導入、カスタマイズ、SDK、セッション形式、プラットフォーム設定への入口をまとめています。
- [quickstart.md](docs/quickstart.md): インストール、認証、最初のセッション開始、基本操作、非対話モードまでを最短で案内します。
- [usage.md](docs/usage.md): 対話 UI、スラッシュコマンド、メッセージキュー、セッション、CLI オプション、設計思想をまとめています。
- [providers.md](docs/providers.md): 認証方式、`auth.json`、環境変数、クラウドプロバイダ設定、優先順位を整理したページです。
- [settings.md](docs/settings.md): 設定項目全体、プロジェクトオーバーライド、思考・UI・セッション・シェル関連の調整方法を扱います。
- [keybindings.md](docs/keybindings.md): 既定ショートカット一覧、キー形式、設定例、Windows/WSL の注意点をまとめています。
- [sessions.md](docs/sessions.md): セッション保存、再開、命名、分岐、`/tree`、`/fork`、`/clone` の運用を説明します。
- [compaction.md](docs/compaction.md): コンテキスト圧縮とブランチ要約の発火条件、構造、設定項目を説明します。

## Customization

- [extensions.md](docs/extensions.md): TypeScript 拡張によるイベント購読、カスタムツール、コマンド、UI、状態管理の拡張方法です。
- [skills.md](docs/skills.md): スキルの配置場所、読み込みの仕組み、`SKILL.md` 構造、frontmatter 要件、検証ルールを説明します。
- [prompt-templates.md](docs/prompt-templates.md): プロンプトテンプレートの保存場所、frontmatter、引数展開、読み込みルールを説明します。
- [themes.md](docs/themes.md): テーマ JSON の配置、選択、必須カラートークン、色形式、設計上のコツをまとめています。
- [packages.md](docs/packages.md): pi package の取得元、構造、依存関係、読み込みフィルタ、スコープ競合解決を説明します。
- [models.md](docs/models.md): `models.json` で独自モデルを追加する方法、互換性フラグ、思考レベルマップ、上書き動作を扱います。
- [custom-provider.md](docs/custom-provider.md): 既存プロバイダの上書き、新規プロバイダ登録、OAuth、独自ストリーミング API 実装を説明します。

## Programmatic Usage

- [sdk.md](docs/sdk.md): Node.js から Pi を埋め込むための SDK と主要 API を説明します。
- [rpc.md](docs/rpc.md): stdin/stdout の JSONL プロトコルで Pi をヘッドレス運用する RPC Mode の仕様です。
- [json.md](docs/json.md): JSON Event Stream Mode のイベント種別、出力形式、実行例をまとめています。
- [tui.md](docs/tui.md): 拡張向けの TUI コンポーネント API、フォーカス制御、テーマ適用、典型パターンを説明します。

## Reference

- [session-format.md](docs/session-format.md): JSONL セッションファイル形式、エントリ型、ツリー構造、SessionManager API の定義です。

## Platform Setup

- [windows.md](docs/windows.md): Windows 上で必要な bash と `shellPath` の設定方法です。
- [termux.md](docs/termux.md): Android/Termux 上のセットアップ、制約、`AGENTS.md` 例、トラブルシュートです。
- [tmux.md](docs/tmux.md): `extended-keys-format csi-u` を前提にした tmux 推奨設定です。
- [terminal-setup.md](docs/terminal-setup.md): Kitty protocol を前提にした主要ターミナルごとの設定方法です。
- [shell-aliases.md](docs/shell-aliases.md): 非対話 bash 実行時にシェルエイリアスを展開する設定例です。

## Development

- [development.md](docs/development.md): ローカル開発セットアップ、fork / リブランディング、デバッグ、テスト、プロジェクト構造を説明します。

## Examples

この節は `examples/` 配下の実装サンプル索引です。ドキュメント本文から参照されるファイルだけでなく、関連する設定ファイル、README、補助スクリプト、生成物も含めて一覧化しています。

### RPC Example

- [rpc-extension-ui.ts](examples/rpc-extension-ui.ts): Pi を RPC モードで起動し、拡張 UI 要求を処理する軽量 TUI クライアントを自前実装する例です。

### SDK Examples

- [sdk/01-minimal.ts](examples/sdk/01-minimal.ts): 既定設定だけで `createAgentSession()` を起動し、出力購読とプロンプト送信を行う最小 SDK 例です。
- [sdk/02-custom-model.ts](examples/sdk/02-custom-model.ts): 組み込みモデルや `models.json` 由来のカスタムモデルを検索し、thinking level を指定してセッションを起動する SDK 例です。
- [sdk/03-custom-prompt.ts](examples/sdk/03-custom-prompt.ts): システムプロンプトを完全置換する方法と既定プロンプトへ指示を追記する方法を比較して示す SDK 例です。
- [sdk/04-skills.ts](examples/sdk/04-skills.ts): SDK でスキルを発見・絞り込み・追加差し替えしつつセッションへ反映する例です。
- [sdk/05-tools.ts](examples/sdk/05-tools.ts): SDK で `createAgentSession()` を使う際に、読み取り専用やカスタム `cwd` などのツール構成を選ぶ方法を示す例です。
- [sdk/06-extensions.ts](examples/sdk/06-extensions.ts): `DefaultResourceLoader` で拡張を追加登録し、インライン拡張や独自ツール/コマンドを組み込む SDK 例です。
- [sdk/07-context-files.ts](examples/sdk/07-context-files.ts): `AGENTS.md` などのコンテキストファイルを発見・上書きし、読み込んだ状態でセッションを生成する例です。
- [sdk/08-prompt-templates.ts](examples/sdk/08-prompt-templates.ts): 仮想テンプレートを追加しつつ `.pi/prompts` からの検出結果も列挙する prompt template 拡張の SDK 例です。
- [sdk/09-api-keys-and-oauth.ts](examples/sdk/09-api-keys-and-oauth.ts): 認証ストレージとモデルレジストリを切り替え、API キーや OAuth 設定の解決方法を示す SDK 例です。
- [sdk/10-settings.ts](examples/sdk/10-settings.ts): `SettingsManager` で設定を上書き・永続化・メモリ内管理しながらセッションへ適用する方法を示す例です。
- [sdk/11-sessions.ts](examples/sdk/11-sessions.ts): メモリ内、新規保存、最近再開、特定セッション再オープンなど `SessionManager` の主要モードを示す例です。
- [sdk/12-full-control.ts](examples/sdk/12-full-control.ts): 認証・モデル・設定・リソースローダー・利用ツールをすべて明示指定してエージェントを完全構成する SDK 例です。
- [sdk/13-session-runtime.ts](examples/sdk/13-session-runtime.ts): `AgentSessionRuntime` でセッションを新規作成・切替した後に購読や拡張バインドを張り直す実装例です。
- [sdk/README.md](examples/sdk/README.md): `createAgentSession()` 系 API を使う SDK サンプル群の目的、実行方法、主要オプションをまとめた案内です。

### Extension Examples

#### Top-level Extensions

- [extensions/README.md](examples/extensions/README.md): Pi の拡張機能サンプル群をカテゴリ別に一覧化し、用途と導入方法を示す索引です。
- [extensions/auto-commit-on-exit.ts](examples/extensions/auto-commit-on-exit.ts): セッション終了時に未コミット変更を自動で `git add` と `git commit` し、直前のアシスタント応答からコミットメッセージを作る例です。
- [extensions/bash-spawn-hook.ts](examples/extensions/bash-spawn-hook.ts): Bash ツール実行前に `source ~/.profile` や環境変数付与を差し込む `spawnHook` の使い方を示す例です。
- [extensions/bookmark.ts](examples/extensions/bookmark.ts): 直近の assistant メッセージにラベルを付けたり外したりして `/tree` で辿りやすくするブックマーク拡張の例です。
- [extensions/border-status-editor.ts](examples/extensions/border-status-editor.ts): エディタの上下ボーダーにスピナー、モデル、コンテキスト使用率、作業ディレクトリ、Git ブランチを表示するカスタム UI 拡張例です。
- [extensions/built-in-tool-renderer.ts](examples/extensions/built-in-tool-renderer.ts): 組み込みの `read` `bash` `edit` `write` ツールの挙動を保ったまま表示だけをコンパクトに差し替える例です。
- [extensions/claude-rules.ts](examples/extensions/claude-rules.ts): `.claude/rules/` 配下の Markdown ルールを走査し、利用可能な規約一覧をシステムプロンプトへ注入する例です。
- [extensions/commands.ts](examples/extensions/commands.ts): 現在のセッションで使えるスラッシュコマンドを列挙し、出所別に絞り込む `/commands` 拡張の例です。
- [extensions/confirm-destructive.ts](examples/extensions/confirm-destructive.ts): セッション削除・切替・分岐の前に確認ダイアログを出し、破壊的操作をキャンセルできるイベントフックの例です。
- [extensions/custom-compaction.ts](examples/extensions/custom-compaction.ts): 会話全体を別モデルで要約して既定の compaction を置き換えるカスタム要約フックの例です。
- [extensions/custom-footer.ts](examples/extensions/custom-footer.ts): Git ブランチ名とトークン使用量を含む独自フッターを `ctx.ui.setFooter()` で描画する例です。
- [extensions/custom-header.ts](examples/extensions/custom-header.ts): TUI の標準ヘッダーを Pi マスコット付きの独自ヘッダーへ差し替え、元に戻すコマンドも提供する例です。
- [extensions/dirty-repo-guard.ts](examples/extensions/dirty-repo-guard.ts): 未コミット変更がある Git リポジトリでセッション切替や fork を確認付きで抑止する例です。
- [extensions/dynamic-tools.ts](examples/extensions/dynamic-tools.ts): セッション開始後や `/add-echo-tool` コマンド経由でツールを動的登録できることを示す例です。
- [extensions/event-bus.ts](examples/extensions/event-bus.ts): 拡張間イベントバス `pi.events` で通知を送受信し、`/emit` コマンドで発火する方法を示す例です。
- [extensions/file-trigger.ts](examples/extensions/file-trigger.ts): 監視中のトリガーファイルへ書き込まれた内容を会話へ注入し、自動で応答ターンを起動する例です。
- [extensions/git-checkpoint.ts](examples/extensions/git-checkpoint.ts): 各ターンで `git stash create` によるチェックポイントを保存し、`/fork` 時に復元可否を選ばせる拡張例です。
- [extensions/git-merge-and-resolve.ts](examples/extensions/git-merge-and-resolve.ts): 各ターン後に upstream を fetch・merge し、競合箇所を追跡して後続メッセージで解決を促す例です。
- [extensions/github-issue-autocomplete.ts](examples/extensions/github-issue-autocomplete.ts): GitHub リポジトリの open issue を事前取得し、`#123` 形式の入力補完をローカルで高速に行う例です。
- [extensions/handoff.ts](examples/extensions/handoff.ts): 現在の会話や compaction 要約を基に新しいセッション向けの引き継ぎプロンプトを生成する `/handoff` 拡張です。
- [extensions/hello.ts](examples/extensions/hello.ts): 名前を受け取って挨拶を返す最小構成のカスタムツール定義例です。
- [extensions/hidden-thinking-label.ts](examples/extensions/hidden-thinking-label.ts): 折りたたまれた thinking ブロックのラベルをコマンドで変更・リセットする UI 拡張例です。
- [extensions/inline-bash.ts](examples/extensions/inline-bash.ts): ユーザープロンプト中の `!{command}` を事前実行して出力へ展開する入力変換フックの例です。
- [extensions/input-transform-streaming.ts](examples/extensions/input-transform-streaming.ts): ユーザー入力に `git diff --stat` を付加して文脈補強しつつ、ストリーミング中の steer では低遅延を優先して変換を省く例です。
- [extensions/input-transform.ts](examples/extensions/input-transform.ts): `input` イベントでユーザー入力を変換したり即時応答で処理打ち切りしたりする方法を示す例です。
- [extensions/interactive-shell.ts](examples/extensions/interactive-shell.ts): `vim` や `git rebase -i` などの対話的コマンド実行時に TUI を一時停止して端末を明け渡す拡張の例です。
- [extensions/mac-system-theme.ts](examples/extensions/mac-system-theme.ts): macOS のダークモード状態を監視し、Pi のテーマを自動で light/dark に同期する拡張例です。
- [extensions/message-renderer.ts](examples/extensions/message-renderer.ts): カスタムメッセージ型に専用レンダラーを登録し、色付き表示や展開時詳細を実装する例です。
- [extensions/minimal-mode.ts](examples/extensions/minimal-mode.ts): 組み込みツールの描画を差し替えて、呼び出しだけを見せる最小表示モードと詳細表示モードを切り替える例です。
- [extensions/modal-editor.ts](examples/extensions/modal-editor.ts): カスタムエディタで Vim 風の normal/insert モード操作を実装する例です。
- [extensions/model-status.ts](examples/extensions/model-status.ts): モデル切替イベントを拾って通知を出し、ステータスバーへ現在モデル名を表示する例です。
- [extensions/notify.ts](examples/extensions/notify.ts): 応答完了時に端末プロトコルや Windows Toast を使ってネイティブ通知を送る拡張例です。
- [extensions/overlay-qa-tests.ts](examples/extensions/overlay-qa-tests.ts): アンカー、マージン、重なり、ストリーミングなどオーバーレイ挙動を網羅的に検証する QA 用拡張です。
- [extensions/overlay-test.ts](examples/extensions/overlay-test.ts): オーバーレイ UI 上でインライン入力、全角文字、絵文字、装飾文字列の重ね描画を検証するテスト画面の例です。
- [extensions/permission-gate.ts](examples/extensions/permission-gate.ts): 危険な bash コマンドを正規表現で検出し、実行前に確認または遮断する安全ゲートの例です。
- [extensions/pirate.ts](examples/extensions/pirate.ts): コマンドで海賊モードを切り替え、`before_agent_start` でシステムプロンプトを差し替えて口調を変える例です。
- [extensions/preset.ts](examples/extensions/preset.ts): モデル、thinking level、ツール、追加指示を JSON プリセットとして切り替える拡張の実装例です。
- [extensions/prompt-customizer.ts](examples/extensions/prompt-customizer.ts): 有効なツールやスキル構成を見てシステムプロンプトへ状況依存の案内を追加する拡張の例です。
- [extensions/protected-paths.ts](examples/extensions/protected-paths.ts): `.env` や `.git` など保護対象パスへの `write` / `edit` ツール呼び出しを遮断する拡張例です。
- [extensions/provider-payload.ts](examples/extensions/provider-payload.ts): プロバイダー送受信のペイロードとレスポンスヘッダーをログへ記録する監査用フックの例です。
- [extensions/qna.ts](examples/extensions/qna.ts): 直前のアシスタント応答から未回答の質問だけを抽出し、ユーザーが追記できる形でエディタへ流し込む例です。
- [extensions/question.ts](examples/extensions/question.ts): 選択肢一覧と自由入力欄を備えた対話的 `question` ツールを自前 UI で実装する例です。
- [extensions/questionnaire.ts](examples/extensions/questionnaire.ts): 単一または複数の質問をタブ UI 付きで提示し、選択肢や自由入力をまとめて回収するツールの例です。
- [extensions/rainbow-editor.ts](examples/extensions/rainbow-editor.ts): 入力中の `ultrathink` 文字列に虹色のアニメーション装飾を付けるカスタムエディタの例です。
- [extensions/reload-runtime.ts](examples/extensions/reload-runtime.ts): コマンドやツールから拡張・スキル・テーマのランタイム再読み込みを安全に起動する例です。
- [extensions/rpc-demo.ts](examples/extensions/rpc-demo.ts): RPC UI プロトコルの `select`、`confirm`、`input`、`editor`、`setStatus` など各種 UI API を一通り試す拡張例です。
- [extensions/send-user-message.ts](examples/extensions/send-user-message.ts): 拡張から `pi.sendUserMessage()` を使って通常送信、steer、follow-up の各ユーザーメッセージ配送を行う例です。
- [extensions/session-name.ts](examples/extensions/session-name.ts): `setSessionName` と `getSessionName` でセッションに表示名を付ける `/session-name` コマンド例です。
- [extensions/shutdown-command.ts](examples/extensions/shutdown-command.ts): コマンドやツールの完了後に `ctx.shutdown()` を呼んで Pi を安全に終了させる方法を示す例です。
- [extensions/snake.ts](examples/extensions/snake.ts): `/snake` で遊べるスネークゲームを実装し、保存・再開・ハイスコア管理も示す TUI 拡張例です。
- [extensions/space-invaders.ts](examples/extensions/space-invaders.ts): キー押下・離上イベントを使って端末上で Space Invaders を遊べるゲーム拡張の例です。
- [extensions/ssh.ts](examples/extensions/ssh.ts): `--ssh` 指定時に read/write/edit/bash と `!` 実行を SSH 先へ委譲し、リモート作業ディレクトリとして振る舞わせる例です。
- [extensions/status-line.ts](examples/extensions/status-line.ts): フッターのステータスラインにターン進行状況を常時表示する `ctx.ui.setStatus()` 利用例です。
- [extensions/structured-output.ts](examples/extensions/structured-output.ts): 終端ツールとして構造化された見出し・要約・アクション項目を返し、追加の LLM ターンなしで完了させる例です。
- [extensions/summarize.ts](examples/extensions/summarize.ts): 現在の会話履歴を抽出してモデルに要約させ、専用 UI で表示する `/summarize` コマンド拡張です。
- [extensions/system-prompt-header.ts](examples/extensions/system-prompt-header.ts): 有効なシステムプロンプト長をステータス表示し `ctx.getSystemPrompt()` 利用を示す例です。
- [extensions/tic-tac-toe.ts](examples/extensions/tic-tac-toe.ts): 三目並べを題材に、ツール呼び出しを `executionMode: "sequential"` で逐次実行する必要性を示す大型デモです。
- [extensions/timed-confirm.ts](examples/extensions/timed-confirm.ts): タイムアウト付き confirm/select ダイアログと `AbortSignal` による手動制御を示す例です。
- [extensions/titlebar-spinner.ts](examples/extensions/titlebar-spinner.ts): エージェント稼働中だけ端末タイトルに点字スピナーを表示して作業状態を示す拡張の例です。
- [extensions/todo.ts](examples/extensions/todo.ts): セッション履歴に状態を保持する `todo` ツールと `/todos` 表示 UI でブランチ対応の TODO 管理を行う例です。
- [extensions/tool-override.ts](examples/extensions/tool-override.ts): `read` ツールを上書きしてアクセス監査、機密パス遮断、ログ閲覧コマンドを追加する例です。
- [extensions/tools.ts](examples/extensions/tools.ts): `/tools` で有効なツール集合を対話的に切り替え、その選択をセッション分岐をまたいで保持する例です。
- [extensions/trigger-compact.ts](examples/extensions/trigger-compact.ts): トークン使用量が閾値を超えたときやコマンド実行時に compaction を起動する拡張例です。
- [extensions/truncated-tool.ts](examples/extensions/truncated-tool.ts): `rg` ラッパーツールで大きな出力を既定サイズに切り詰め、全文を一時ファイルへ退避する実装例です。
- [extensions/widget-placement.ts](examples/extensions/widget-placement.ts): エディタの上部と下部にウィジェットを配置する最小構成の UI 配置例です。
- [extensions/working-indicator.ts](examples/extensions/working-indicator.ts): ストリーミング中の作業インジケータをドット、パルス、非表示、独自スピナーへ切り替える拡張例です。
- [extensions/working-message-test.ts](examples/extensions/working-message-test.ts): カスタムの作業中メッセージとインジケーターがセッションを跨いで維持されるか確認する例です。

#### Multi-file Extension Packages

- [extensions/custom-provider-anthropic/](examples/extensions/custom-provider-anthropic): OAuth と API キーの両対応で Anthropic 互換プロバイダーを登録し、関連設定ファイルまで含めて構成を示す複数ファイル例です。
- [extensions/custom-provider-gitlab-duo/](examples/extensions/custom-provider-gitlab-duo): GitLab Duo の AI Gateway を Anthropic/OpenAI 互換として扱うカスタムプロバイダー一式です。
- [extensions/doom-overlay/](examples/extensions/doom-overlay): DOOM をオーバーレイ UI 上で動かすための TypeScript、C、WASM、ビルドスクリプト、README を含む大型デモです。
- [extensions/dynamic-resources/](examples/extensions/dynamic-resources): `resources_discover` を使ってスキル、プロンプト、テーマを動的公開する拡張と関連リソース一式です。
- [extensions/plan-mode/](examples/extensions/plan-mode): 読み取り専用ツール制限、計画抽出、進捗表示をまとめた Plan Mode 拡張一式です。
- [extensions/sandbox/](examples/extensions/sandbox): 組み込み `bash` をサンドボックス実行へ差し替える拡張本体と依存設定一式です。
- [extensions/subagent/](examples/extensions/subagent): subagent 定義、委譲ツール実装、役割別エージェント設定、チェーン用プロンプトをまとめた複数ファイル例です。
- [extensions/with-deps/](examples/extensions/with-deps): 拡張専用 `node_modules` を解決しながら外部依存を使う最小構成パッケージ例です。

## 原典

- Docs トップ: `https://pi.dev/docs/latest`
- ソース: `https://github.com/earendil-works/pi/tree/main/packages/coding-agent/docs`
