# 設定

> 要約: Pi はグローバル設定とプロジェクト設定の 2 層で JSON 設定を読み込み、プロジェクト側が上書きします。このページはモデル、UI、再試行、コンパクション、リソース読み込みなど、主要な設定キーとその優先順位をまとめたものです。

Pi は JSON 設定ファイルを使い、プロジェクト設定がグローバル設定を上書きします。

| 場所 | スコープ |
|----------|-------|
| `~/.pi/agent/settings.json` | グローバル（すべてのプロジェクト） |
| `.pi/settings.json` | プロジェクト（現在のディレクトリ） |

直接編集するか、一般的なオプションなら `/settings` を使ってください。

## すべての設定

### モデルと Thinking

| 設定 | 型 | デフォルト | 説明 |
|---------|------|---------|-------------|
| `defaultProvider` | string | - | デフォルトのプロバイダ（例: `"anthropic"`, `"openai"`） |
| `defaultModel` | string | - | デフォルトのモデル ID |
| `defaultThinkingLevel` | string | - | `"off"`, `"minimal"`, `"low"`, `"medium"`, `"high"`, `"xhigh"` |
| `hideThinkingBlock` | boolean | `false` | 出力内の thinking ブロックを隠す |
| `thinkingBudgets` | object | - | thinking レベルごとのカスタムトークン予算 |

#### thinkingBudgets

```json
{
  "thinkingBudgets": {
    "minimal": 1024,
    "low": 4096,
    "medium": 10240,
    "high": 32768
  }
}
```

### UI と表示

| 設定 | 型 | デフォルト | 説明 |
|---------|------|---------|-------------|
| `theme` | string | `"dark"` | テーマ名（`"dark"`, `"light"`, またはカスタム） |
| `quietStartup` | boolean | `false` | 起動時ヘッダーを非表示にする |
| `collapseChangelog` | boolean | `false` | 更新後に圧縮された changelog を表示する |
| `enableInstallTelemetry` | boolean | `true` | 初回インストール後、または changelog で更新検出後に匿名のインストール / 更新バージョン ping を送る。更新確認自体の制御ではない |
| `doubleEscapeAction` | string | `"tree"` | Escape を 2 回押したときの動作: `"tree"`, `"fork"`, `"none"` |
| `treeFilterMode` | string | `"default"` | `/tree` のデフォルトフィルタ: `"default"`, `"no-tools"`, `"user-only"`, `"labeled-only"`, `"all"` |
| `editorPaddingX` | number | `0` | 入力エディタの左右パディング（0-3） |
| `autocompleteMaxVisible` | number | `5` | オートコンプリート候補の最大表示数（3-20） |
| `showHardwareCursor` | boolean | `false` | IME 対応のため TUI が位置決めしている間も端末カーソルを表示する |

### テレメトリと更新確認

`enableInstallTelemetry` は `https://pi.dev/api/report-install` への匿名インストール / 更新 ping だけを制御します。テレメトリを無効化しても更新確認は止まりません。Pi は最新バージョン確認のために `https://pi.dev/api/latest-version` を取得することがあります。

Pi のバージョン更新確認を無効化するには `PI_SKIP_VERSION_CHECK=1` を設定してください。ここで説明している更新確認、パッケージ更新確認、インストール / 更新テレメトリを含むすべての起動時ネットワーク処理を無効化するには `--offline` または `PI_OFFLINE=1` を使います。

### 警告

| 設定 | 型 | デフォルト | 説明 |
|---------|------|---------|-------------|
| `warnings.anthropicExtraUsage` | boolean | `true` | Anthropic のサブスクリプション認証が有料の追加利用を使う可能性があるときに警告を表示する |

```json
{
  "warnings": {
    "anthropicExtraUsage": false
  }
}
```

### コンパクション

| 設定 | 型 | デフォルト | 説明 |
|---------|------|---------|-------------|
| `compaction.enabled` | boolean | `true` | 自動コンパクションを有効にする |
| `compaction.reserveTokens` | number | `16384` | LLM 応答用に予約するトークン数 |
| `compaction.keepRecentTokens` | number | `20000` | 保持する最近のトークン数（要約しない） |

```json
{
  "compaction": {
    "enabled": true,
    "reserveTokens": 16384,
    "keepRecentTokens": 20000
  }
}
```

### ブランチ要約

| 設定 | 型 | デフォルト | 説明 |
|---------|------|---------|-------------|
| `branchSummary.reserveTokens` | number | `16384` | ブランチ要約用に予約するトークン数 |
| `branchSummary.skipPrompt` | boolean | `false` | `/tree` ナビゲーション時の「Summarize branch?」プロンプトをスキップする（デフォルトでは要約しない） |

### 再試行

| 設定 | 型 | デフォルト | 説明 |
|---------|------|---------|-------------|
| `retry.enabled` | boolean | `true` | 一時的エラーに対するエージェントレベルの自動再試行を有効にする |
| `retry.maxRetries` | number | `3` | エージェントレベル再試行の最大回数 |
| `retry.baseDelayMs` | number | `2000` | エージェントレベル指数バックオフの基準遅延（2s, 4s, 8s） |
| `retry.provider.timeoutMs` | number | SDK default | プロバイダ / SDK リクエストのタイムアウト（ミリ秒） |
| `retry.provider.maxRetries` | number | `0` | プロバイダ / SDK 側の再試行回数 |
| `retry.provider.maxRetryDelayMs` | number | `60000` | 失敗と判断する前に許容する、サーバー要求ベースの最大遅延（60 秒） |

プロバイダが `retry.provider.maxRetryDelayMs` より長い再試行遅延を要求した場合（例: Google の「quota will reset after 5h」）、そのリクエストは黙って待機せず、説明付きエラーで即座に失敗します。上限を無効にするには `0` を設定してください。

`retry.provider.maxRetries` は、プロバイダレベルの再試行が明示的に必要でない限り `0` のままにしてください。これを `0` より大きくすると、Pi が検知する前に SDK / プロバイダ側の再試行が利用量制限エラーを処理してしまい、状況によってはプロバイダのクォータが戻るまでエージェントが止まる可能性があります。

```json
{
  "retry": {
    "enabled": true,
    "maxRetries": 3,
    "baseDelayMs": 2000,
    "provider": {
      "timeoutMs": 3600000,
      "maxRetries": 0,
      "maxRetryDelayMs": 60000
    }
  }
}
```

### メッセージ配信

| 設定 | 型 | デフォルト | 説明 |
|---------|------|---------|-------------|
| `steeringMode` | string | `"one-at-a-time"` | ステアリングメッセージの送信方法: `"all"` または `"one-at-a-time"` |
| `followUpMode` | string | `"one-at-a-time"` | フォローアップメッセージの送信方法: `"all"` または `"one-at-a-time"` |
| `transport` | string | `"auto"` | 複数のトランスポートをサポートするプロバイダ向けの優先トランスポート: `"sse"`, `"websocket"`, `"websocket-cached"`, `"auto"` |
| `httpIdleTimeoutMs` | number | `300000` | HTTP ヘッダー / ボディのアイドルタイムアウト（ミリ秒）。明示的なストリームアイドルタイムアウトを持つプロバイダでも使われる。無効化は `0` |
| `websocketConnectTimeoutMs` | number | `15000` | WebSocket 接続 / オープンハンドシェイクのタイムアウト（ミリ秒）。WebSocket トランスポート対応プロバイダ向け。無効化は `0` |

### ターミナルと画像

| 設定 | 型 | デフォルト | 説明 |
|---------|------|---------|-------------|
| `terminal.showImages` | boolean | `true` | 端末内に画像を表示する（対応環境のみ） |
| `terminal.imageWidthCells` | number | `60` | 端末セル単位でのインライン画像の推奨幅 |
| `terminal.clearOnShrink` | boolean | `false` | 内容が縮んだときに空行を消去する（ちらつきの原因になることがある） |
| `images.autoResize` | boolean | `true` | 画像を最大 2000x2000 にリサイズする |
| `images.blockImages` | boolean | `false` | すべての画像を LLM へ送らないようにする |

### シェル

| 設定 | 型 | デフォルト | 説明 |
|---------|------|---------|-------------|
| `shellPath` | string | - | カスタムシェルパス（例: Windows の Cygwin 用） |
| `shellCommandPrefix` | string | - | すべての bash コマンドに付与するプレフィックス（例: `"shopt -s expand_aliases"`） |
| `npmCommand` | string[] | - | npm パッケージの検索 / インストールに使うコマンド argv（例: `["mise", "exec", "node@20", "--", "npm"]`） |

```json
{
  "npmCommand": ["mise", "exec", "node@20", "--", "npm"]
}
```

`npmCommand` は、インストール、アンインストール、git パッケージ内の依存インストールを含むすべての npm パッケージマネージャ操作で使われます。ユーザースコープの npm パッケージは `~/.pi/agent/npm/` に、プロジェクトスコープの npm パッケージは `.pi/npm/` にインストールされます。argv 形式で、プロセスがそのまま起動される形を正確に指定してください。`npmCommand` が設定されている場合、git パッケージの依存インストールではラッパーや代替パッケージマネージャに npm 固有フラグを流さないよう、単純な `install` が使われます。

### セッション

| 設定 | 型 | デフォルト | 説明 |
|---------|------|---------|-------------|
| `sessionDir` | string | - | セッションファイルの保存先ディレクトリ。絶対パス、相対パス、`~` を受け付ける |

```json
{ "sessionDir": ".pi/sessions" }
```

複数の場所でセッションディレクトリが指定された場合、優先順位は `--session-dir`、`PI_CODING_AGENT_SESSION_DIR`、`settings.json` 内の `sessionDir` の順です。

### モデル切り替え

| 設定 | 型 | デフォルト | 説明 |
|---------|------|---------|-------------|
| `enabledModels` | string[] | - | `Ctrl+P` によるモデル巡回用のモデルパターン（`--models` CLI フラグと同形式） |

```json
{
  "enabledModels": ["claude-*", "gpt-4o", "gemini-2*"]
}
```

### Markdown

| 設定 | 型 | デフォルト | 説明 |
|---------|------|---------|-------------|
| `markdown.codeBlockIndent` | string | `"  "` | コードブロックのインデント |

### リソース

これらの設定は、拡張機能、スキル、プロンプト、テーマをどこから読み込むかを定義します。

`~/.pi/agent/settings.json` 内のパスは `~/.pi/agent` 基準で解決されます。`.pi/settings.json` 内のパスは `.pi` 基準で解決されます。絶対パスと `~` もサポートされます。

| 設定 | 型 | デフォルト | 説明 |
|---------|------|---------|-------------|
| `packages` | array | `[]` | リソースを読み込む npm / git パッケージ |
| `extensions` | string[] | `[]` | ローカル拡張ファイルパスまたはディレクトリ |
| `skills` | string[] | `[]` | ローカルスキルファイルパスまたはディレクトリ |
| `prompts` | string[] | `[]` | ローカルプロンプトテンプレートのパスまたはディレクトリ |
| `themes` | string[] | `[]` | ローカルテーマファイルパスまたはディレクトリ |
| `enableSkillCommands` | boolean | `true` | スキルを `/skill:name` コマンドとして登録する |

配列は glob パターンと除外をサポートします。`!pattern` で除外します。`+path` で完全一致のパスを強制的に含め、`-path` で完全一致のパスを強制的に除外します。

#### packages

文字列形式では、1 つのパッケージからすべてのリソースを読み込みます。

```json
{
  "packages": ["pi-skills", "@org/my-extension"]
}
```

オブジェクト形式では、どのリソースを読み込むかを絞り込めます。

```json
{
  "packages": [
    {
      "source": "pi-skills",
      "skills": ["brave-search", "transcribe"],
      "extensions": []
    }
  ]
}
```

パッケージ管理の詳細は [packages.md](packages.md) を参照してください。

## 例

```json
{
  "defaultProvider": "anthropic",
  "defaultModel": "claude-sonnet-4-20250514",
  "defaultThinkingLevel": "medium",
  "theme": "dark",
  "compaction": {
    "enabled": true,
    "reserveTokens": 16384,
    "keepRecentTokens": 20000
  },
  "retry": {
    "enabled": true,
    "maxRetries": 3
  },
  "enabledModels": ["claude-*", "gpt-4o"],
  "warnings": {
    "anthropicExtraUsage": true
  },
  "packages": ["pi-skills"]
}
```

## プロジェクトによる上書き

プロジェクト設定（`.pi/settings.json`）はグローバル設定を上書きします。ネストしたオブジェクトはマージされます。

```json
// ~/.pi/agent/settings.json (global)
{
  "theme": "dark",
  "compaction": { "enabled": true, "reserveTokens": 16384 }
}

// .pi/settings.json (project)
{
  "compaction": { "reserveTokens": 8192 }
}

// Result
{
  "theme": "dark",
  "compaction": { "enabled": true, "reserveTokens": 8192 }
}
```
