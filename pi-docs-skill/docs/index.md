# Pi ドキュメント

> 要約: Pi は最小限のコアを持つターミナル向けコーディングハーネスで、拡張・スキル・プロンプトテンプレート・テーマ・pi package によって機能を広げます。このページは、導入からカスタマイズ、SDK、セッション形式、各種プラットフォーム設定までの入口をまとめた総合目次です。

Pi は最小限のターミナル向けコーディングハーネスです。コアは小さく保ちつつ、TypeScript 拡張、スキル、プロンプトテンプレート、テーマ、pi package を通じて拡張できるよう設計されています。

## クイックスタート

npm で Pi をインストールします。

```bash
npm install -g --ignore-scripts @earendil-works/pi-coding-agent
```

`--ignore-scripts` は、インストール時に依存パッケージのライフサイクルスクリプトを無効にします。通常の npm インストールでは、Pi にインストールスクリプトは不要です。

Linux または macOS では、インストーラも利用できます。

```bash
curl -fsSL https://pi.dev/install.sh | sh
```

Pi 自体をアンインストールするには、curl インストールと npm インストールのどちらでも npm を使います。

```bash
npm uninstall -g @earendil-works/pi-coding-agent
```

pnpm、Yarn、Bun でインストールした場合は、それぞれ対応するグローバル削除コマンドを使ってください: `pnpm remove -g @earendil-works/pi-coding-agent`、`yarn global remove @earendil-works/pi-coding-agent`、`bun uninstall -g @earendil-works/pi-coding-agent`。

その後、プロジェクトディレクトリで実行します。

```bash
pi
```

サブスクリプション型プロバイダを使う場合は `/login` で認証し、API キー型プロバイダを使う場合は `ANTHROPIC_API_KEY` のような環境変数を設定してから Pi を起動します。

初回起動の完全な手順は [Quickstart](quickstart.md) を参照してください。

## ここから始める

- [Quickstart](quickstart.md) - インストール、認証、最初のセッションの実行。
- [Using Pi](usage.md) - 対話モード、スラッシュコマンド、コンテキストファイル、CLI リファレンス。
- [Providers](providers.md) - 組み込みプロバイダ向けのサブスクリプション設定と API キー設定。
- [Settings](settings.md) - グローバル設定とプロジェクト設定。
- [Keybindings](keybindings.md) - 既定ショートカットとカスタムキーバインド。
- [Sessions](sessions.md) - セッション管理、分岐、ツリーナビゲーション。
- [Compaction](compaction.md) - コンテキスト圧縮とブランチ要約。

## カスタマイズ

- [Extensions](extensions.md) - ツール、コマンド、イベント、カスタム UI を追加する TypeScript モジュール。
- [Skills](skills.md) - 再利用可能なオンデマンド能力としての Agent Skill。
- [Prompt templates](prompt-templates.md) - スラッシュコマンドから展開できる再利用可能プロンプト。
- [Themes](themes.md) - 組み込みおよびカスタムのターミナルテーマ。
- [Pi packages](packages.md) - 拡張、スキル、プロンプト、テーマを束ねて共有する仕組み。
- [Custom models](models.md) - 対応プロバイダ API 向けにモデル定義を追加。
- [Custom providers](custom-provider.md) - 独自 API と OAuth フローを実装。

## プログラムからの利用

- [SDK](sdk.md) - Node.js アプリケーションに Pi を組み込む。
- [RPC mode](rpc.md) - stdin/stdout の JSONL を介した統合。
- [JSON event stream mode](json.md) - 構造化イベント付きの print モード。
- [TUI components](tui.md) - 拡張向けのカスタムターミナル UI を構築。

## リファレンス

- [Session format](session-format.md) - JSONL セッションファイル形式、エントリ型、SessionManager API。

## プラットフォーム設定

- [Windows](windows.md)
- [Termux on Android](termux.md)
- [tmux](tmux.md)
- [Terminal setup](terminal-setup.md)
- [Shell aliases](shell-aliases.md)

## 開発

- [Development](development.md) - ローカルセットアップ、プロジェクト構造、デバッグ方法。
