# 開発

> 要約: このページは Pi 本体をソースから開発する人向けのセットアップ、fork 時のブランド変更、アセットパス解決、デバッグ、テスト、パッケージ構成を説明します。開発時に守るべき基準点を短くまとめたページです。

追加のガイドラインは [AGENTS.md](https://github.com/earendil-works/pi-mono/blob/main/AGENTS.md) を参照してください。

## セットアップ

```bash
git clone https://github.com/earendil-works/pi-mono
cd pi-mono
npm install
npm run build
```

ソースから実行するには次を使います。

```bash
/path/to/pi-mono/pi-test.sh
```

このスクリプトは任意のディレクトリから実行できます。Pi は呼び出し元の現在の作業ディレクトリを維持します。

## Fork / リブランディング

`package.json` で設定します。

```json
{
  "piConfig": {
    "name": "pi",
    "configDir": ".pi"
  }
}
```

fork では `name`、`configDir`、`bin` フィールドを変更してください。CLI バナー、設定パス、環境変数名に影響します。

## パス解決

実行モードは 3 種類あります: npm install、スタンドアロンバイナリ、ソースからの tsx 実行です。

パッケージアセットには **常に `src/config.ts` を使ってください**。

```typescript
import { getPackageDir, getThemeDir } from "./config.js";
```

パッケージアセットに対して `__dirname` を直接使ってはいけません。

## デバッグコマンド

非表示の `/debug` は `~/.pi/agent/pi-debug.log` に次を書き込みます。

- ANSI コード付きで描画された TUI 行
- LLM へ送られた直近のメッセージ

## テスト

```bash
./test.sh                         # 非 LLM テストを実行（API キー不要）
npm test                          # 全テストを実行
npm test -- test/specific.test.ts # 特定テストを実行
```

## プロジェクト構造

```text
packages/
  ai/           # LLM provider abstraction
  agent/        # Agent loop and message types
  tui/          # Terminal UI components
  coding-agent/ # CLI and interactive mode
```
