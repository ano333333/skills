# クイックスタート

> 要約: このページは、Pi のインストール、認証、最初のセッション開始までを最短で案内します。最初に試す操作、プロジェクト指示の与え方、セッション再開やワンショット実行の基本もまとまっています。

このページでは、インストールから役に立つ最初の Pi セッション開始までを説明します。

## インストール

Pi は npm パッケージとして配布されています。

```bash
npm install -g --ignore-scripts @earendil-works/pi-coding-agent
```

`--ignore-scripts` は、インストール時に依存パッケージのライフサイクルスクリプトを無効にします。通常の npm インストールでは、Pi にインストールスクリプトは不要です。

### アンインストール

Pi をインストールしたパッケージマネージャを使って削除します。curl インストーラも内部では npm をグローバルに使うため、curl インストールと npm インストールの削除はどちらも npm です。

```bash
# curl インストーラまたは npm install -g
npm uninstall -g @earendil-works/pi-coding-agent

# pnpm
pnpm remove -g @earendil-works/pi-coding-agent

# Yarn
yarn global remove @earendil-works/pi-coding-agent

# Bun
bun uninstall -g @earendil-works/pi-coding-agent
```

Pi をアンインストールしても、`~/.pi/agent/` にある設定、認証情報、セッション、インストール済み pi package は残ります。

その後、Pi を動かしたいプロジェクトディレクトリで起動します。

```bash
cd /path/to/project
pi
```

## 認証

Pi は `/login` によるサブスクリプション型プロバイダ、または環境変数や auth ファイルによる API キー型プロバイダを利用できます。

### 方法 1: サブスクリプションログイン

Pi を起動して次を実行します。

```text
/login
```

その後、プロバイダを選択します。組み込みのサブスクリプションログインには、Claude Pro/Max、ChatGPT Plus/Pro (Codex)、GitHub Copilot があります。

### 方法 2: API キー

Pi を起動する前に API キーを設定します。

```bash
export ANTHROPIC_API_KEY=sk-ant-...
pi
```

`/login` を実行して API キー型プロバイダを選ぶと、キーを `~/.pi/agent/auth.json` に保存することもできます。

対応プロバイダ、環境変数、クラウドプロバイダ設定の一覧は [Providers](providers.md) を参照してください。

## 最初のセッション

Pi が起動したら、リクエストを入力して Enter を押します。

```text
Summarize this repository and tell me how to run its checks.
```

既定では、Pi はモデルに次の 4 つのツールを渡します。

- `read` - ファイルを読む
- `write` - ファイルを新規作成または上書きする
- `edit` - ファイルにパッチを当てる
- `bash` - シェルコマンドを実行する

追加の組み込み読み取り専用ツール (`grep`、`find`、`ls`) もツールオプションから利用できます。Pi は現在の作業ディレクトリ内で動作し、その場所のファイルを変更できます。簡単に巻き戻せるようにしたい場合は、git などのチェックポイント手順を使ってください。

## Pi にプロジェクト指示を与える

Pi は起動時にコンテキストファイルを読み込みます。プロジェクト内でどう動くべきかを伝えるために `AGENTS.md` を追加します。

```markdown
# Project Instructions

- Run `npm run check` after code changes.
- Do not run production migrations locally.
- Keep responses concise.
```

Pi は次を読み込みます。

- グローバル指示として `~/.pi/agent/AGENTS.md`
- 親ディレクトリおよび現在のディレクトリにある `AGENTS.md` または `CLAUDE.md`

コンテキストファイルを変更したあとは、Pi を再起動するか `/reload` を実行してください。

## よく試す操作

### ファイルを参照する

エディタで `@` を入力するとファイルをあいまい検索できます。あるいはコマンドラインでファイルを渡せます。

```bash
pi @README.md "Summarize this"
pi @src/app.ts @src/app.test.ts "Review these together"
```

対応ターミナルでは、画像を Ctrl+V で貼り付けるか、ドラッグ＆ドロップできます。Windows では Alt+V を使います。

### シェルコマンドを実行する

対話モードでは次のように入力します。

```text
!npm run lint
```

コマンド出力はモデルへ送られます。出力をモデルのコンテキストへ追加せずに実行したい場合は `!!command` を使います。

### モデルを切り替える

`/model` または Ctrl+L でモデルを選択します。Shift+Tab で思考レベルを切り替えます。Ctrl+P / Shift+Ctrl+P で scoped model を順送りできます。

### 後で続きから再開する

セッションは自動保存されます。

```bash
pi -c                  # 直近のセッションを続ける
pi -r                  # 過去のセッションを一覧表示して選ぶ
pi --name "my task"    # 起動時にセッション表示名を設定
pi --session <path|id> # 特定のセッションを開く
```

Pi 内では `/resume`、`/new`、`/tree`、`/fork`、`/clone` でセッションを管理します。

### 非対話モード

ワンショットのプロンプトには次を使います。

```bash
pi -p "Summarize this codebase"
cat README.md | pi -p "Summarize this text"
pi -p @screenshot.png "What's in this image?"
```

構造化 JSON イベント出力には `--mode json`、プロセス統合には `--mode rpc` を使います。

## 次のステップ

- [Using Pi](usage.md) - 対話モード、スラッシュコマンド、セッション、コンテキストファイル、CLI リファレンス。
- [Providers](providers.md) - 認証とモデル設定。
- [Settings](settings.md) - グローバル設定とプロジェクト設定。
- [Keybindings](keybindings.md) - ショートカットとカスタマイズ。
- [Pi Packages](packages.md) - 共有用の拡張、スキル、プロンプト、テーマをインストール。

プラットフォーム別の注意事項: [Windows](windows.md)、[Termux](termux.md)、[tmux](tmux.md)、[Terminal setup](terminal-setup.md)、[Shell aliases](shell-aliases.md)。
