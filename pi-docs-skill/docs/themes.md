# テーマ

> 要約: テーマは、Pi の TUI に使われる色を定義する JSON ファイルです。このページでは、配置場所、選択方法、JSON 形式、必須カラートークン、色指定の互換性と設計上のコツをまとめています。

テーマは、TUI の色を定義する JSON ファイルです。

## 目次

- [配置場所](#配置場所)
- [テーマの選択](#テーマの選択)
- [カスタムテーマの作成](#カスタムテーマの作成)
- [テーマ形式](#テーマ形式)
- [カラートークン](#カラートークン)
- [色の値](#色の値)
- [ヒント](#ヒント)

## 配置場所

Pi は次の場所からテーマを読み込みます。

- 組み込み: `dark`, `light`
- グローバル: `~/.pi/agent/themes/*.json`
- プロジェクト: `.pi/themes/*.json`
- パッケージ: `themes/` ディレクトリ、または `package.json` の `pi.themes` エントリ
- Settings: ファイルまたはディレクトリを含む `themes` 配列
- CLI: `--theme <path>`（複数回指定可能）

検出を無効にするには `--no-themes` を使います。

## テーマの選択

テーマは `/settings` または `settings.json` で選択します。

```json
{
  "theme": "my-theme"
}
```

初回実行時、pi はターミナルの背景色を検出し、`dark` または `light` をデフォルトとして選びます。

## カスタムテーマの作成

1. テーマファイルを作成します。

```bash
mkdir -p ~/.pi/agent/themes
vim ~/.pi/agent/themes/my-theme.json
```

2. 必要なすべての色を含めてテーマを定義します（[カラートークン](#カラートークン) を参照）。

```json
{
  "$schema": "https://raw.githubusercontent.com/earendil-works/pi/main/packages/coding-agent/src/modes/interactive/theme/theme-schema.json",
  "name": "my-theme",
  "vars": {
    "primary": "#00aaff",
    "secondary": 242
  },
  "colors": {
    "accent": "primary",
    "border": "primary",
    "borderAccent": "#00ffff",
    "borderMuted": "secondary",
    "success": "#00ff00",
    "error": "#ff0000",
    "warning": "#ffff00",
    "muted": "secondary",
    "dim": 240,
    "text": "",
    "thinkingText": "secondary",
    "selectedBg": "#2d2d30",
    "userMessageBg": "#2d2d30",
    "userMessageText": "",
    "customMessageBg": "#2d2d30",
    "customMessageText": "",
    "customMessageLabel": "primary",
    "toolPendingBg": "#1e1e2e",
    "toolSuccessBg": "#1e2e1e",
    "toolErrorBg": "#2e1e1e",
    "toolTitle": "primary",
    "toolOutput": "",
    "mdHeading": "#ffaa00",
    "mdLink": "primary",
    "mdLinkUrl": "secondary",
    "mdCode": "#00ffff",
    "mdCodeBlock": "",
    "mdCodeBlockBorder": "secondary",
    "mdQuote": "secondary",
    "mdQuoteBorder": "secondary",
    "mdHr": "secondary",
    "mdListBullet": "#00ffff",
    "toolDiffAdded": "#00ff00",
    "toolDiffRemoved": "#ff0000",
    "toolDiffContext": "secondary",
    "syntaxComment": "secondary",
    "syntaxKeyword": "primary",
    "syntaxFunction": "#00aaff",
    "syntaxVariable": "#ffaa00",
    "syntaxString": "#00ff00",
    "syntaxNumber": "#ff00ff",
    "syntaxType": "#00aaff",
    "syntaxOperator": "primary",
    "syntaxPunctuation": "secondary",
    "thinkingOff": "secondary",
    "thinkingMinimal": "primary",
    "thinkingLow": "#00aaff",
    "thinkingMedium": "#00ffff",
    "thinkingHigh": "#ff00ff",
    "thinkingXhigh": "#ff0000",
    "bashMode": "#ffaa00"
  }
}
```

3. `/settings` からテーマを選択します。

**ホットリロード:** 現在有効なカスタムテーマファイルを編集すると、pi は即座の視覚フィードバックのために自動で再読み込みします。

## テーマ形式

```json
{
  "$schema": "https://raw.githubusercontent.com/earendil-works/pi/main/packages/coding-agent/src/modes/interactive/theme/theme-schema.json",
  "name": "my-theme",
  "vars": {
    "blue": "#0066cc",
    "gray": 242
  },
  "colors": {
    "accent": "blue",
    "muted": "gray",
    "text": "",
    ...
  }
}
```

- `name` は必須で、一意でなければなりません。
- `vars` は任意です。再利用したい色をここで定義し、`colors` から参照します。
- `colors` では、必須の 51 個のトークンをすべて定義する必要があります。

`$schema` フィールドを入れると、エディタの自動補完とバリデーションが有効になります。

## カラートークン

すべてのテーマは 51 個のカラートークンを定義する必要があります。任意の色はありません。

### コア UI（11 色）

| Token | 用途 |
|-------|------|
| `accent` | 主要アクセント（ロゴ、選択項目、カーソル） |
| `border` | 通常の境界線 |
| `borderAccent` | 強調された境界線 |
| `borderMuted` | 控えめな境界線（エディタ） |
| `success` | 成功状態 |
| `error` | エラー状態 |
| `warning` | 警告状態 |
| `muted` | 補助テキスト |
| `dim` | 3 次的なテキスト |
| `text` | 既定のテキスト（通常は `""`） |
| `thinkingText` | Thinking ブロック内のテキスト |

### 背景とコンテンツ（11 色）

| Token | 用途 |
|-------|------|
| `selectedBg` | 選択行の背景 |
| `userMessageBg` | ユーザーメッセージ背景 |
| `userMessageText` | ユーザーメッセージ本文 |
| `customMessageBg` | 拡張メッセージ背景 |
| `customMessageText` | 拡張メッセージ本文 |
| `customMessageLabel` | 拡張メッセージラベル |
| `toolPendingBg` | ツールボックス（保留中） |
| `toolSuccessBg` | ツールボックス（成功） |
| `toolErrorBg` | ツールボックス（エラー） |
| `toolTitle` | ツールタイトル |
| `toolOutput` | ツール出力テキスト |

### Markdown（10 色）

| Token | 用途 |
|-------|------|
| `mdHeading` | 見出し |
| `mdLink` | リンクテキスト |
| `mdLinkUrl` | リンク URL |
| `mdCode` | インラインコード |
| `mdCodeBlock` | コードブロック内容 |
| `mdCodeBlockBorder` | コードブロックの囲い |
| `mdQuote` | 引用テキスト |
| `mdQuoteBorder` | 引用の境界線 |
| `mdHr` | 水平線 |
| `mdListBullet` | リストの箇条書き記号 |

### ツール差分（3 色）

| Token | 用途 |
|-------|------|
| `toolDiffAdded` | 追加行 |
| `toolDiffRemoved` | 削除行 |
| `toolDiffContext` | コンテキスト行 |

### シンタックスハイライト（9 色）

| Token | 用途 |
|-------|------|
| `syntaxComment` | コメント |
| `syntaxKeyword` | キーワード |
| `syntaxFunction` | 関数名 |
| `syntaxVariable` | 変数 |
| `syntaxString` | 文字列 |
| `syntaxNumber` | 数値 |
| `syntaxType` | 型 |
| `syntaxOperator` | 演算子 |
| `syntaxPunctuation` | 句読点・記号 |

### Thinking レベルの境界線（6 色）

Thinking レベルを示すエディタ境界線の色です（控えめなものから目立つものへの視覚階層）。

| Token | 用途 |
|-------|------|
| `thinkingOff` | Thinking 無効 |
| `thinkingMinimal` | 最小限の Thinking |
| `thinkingLow` | 低い Thinking |
| `thinkingMedium` | 中程度の Thinking |
| `thinkingHigh` | 高い Thinking |
| `thinkingXhigh` | 非常に高い Thinking |

### Bash モード（1 色）

| Token | 用途 |
|-------|------|
| `bashMode` | bash モード（`!` プレフィックス）のエディタ境界線 |

### HTML エクスポート（任意）

`export` セクションは `/export` の HTML 出力で使う色を制御します。省略した場合、色は `userMessageBg` から導出されます。

```json
{
  "export": {
    "pageBg": "#18181e",
    "cardBg": "#1e1e24",
    "infoBg": "#3c3728"
  }
}
```

## 色の値

4 つの形式がサポートされています。

| Format | 例 | 説明 |
|--------|----|------|
| Hex | `"#ff0000"` | 6 桁の 16 進 RGB |
| 256-color | `39` | xterm 256 色パレットのインデックス（0-255） |
| Variable | `"primary"` | `vars` エントリへの参照 |
| Default | `""` | ターミナルの既定色 |

### 256 色パレット

- `0-15`: 基本 ANSI カラー（ターミナル依存）
- `16-231`: 6×6×6 の RGB キューブ（`16 + 36×R + 6×G + B`。R/G/B は 0-5）
- `232-255`: グレースケールの段階

### ターミナル互換性

Pi は 24-bit RGB カラーを使用します。ほとんどの現代的ターミナルはこれをサポートしています（iTerm2、Kitty、WezTerm、Windows Terminal、VS Code）。256 色しかサポートしない古いターミナルでは、pi はもっとも近い色へフォールバックします。

truecolor サポートの確認:

```bash
echo $COLORTERM  # "truecolor" または "24bit" が出力されるはずです
```

## ヒント

**暗いターミナル:** コントラストが高く、明るく鮮やかな色を使います。

**明るいターミナル:** 低めのコントラストで、より暗く落ち着いた色を使います。

**色の調和:** ベースとなるパレット（Nord、Gruvbox、Tokyo Night など）から始め、`vars` に定義して一貫して参照します。

**テスト:** 異なるメッセージ種別、ツール状態、Markdown コンテンツ、長く折り返されるテキストでテーマを確認してください。

**VS Code:** 正確な色表示のために `terminal.integrated.minimumContrastRatio` を `1` に設定します。

## 例

組み込みテーマを参照してください。
- [dark.json](../src/modes/interactive/theme/dark.json)
- [light.json](../src/modes/interactive/theme/light.json)
