# 拡張機能

> 要約: Extensions は、pi の挙動を TypeScript モジュールで拡張する仕組みです。ライフサイクルイベントの購読、LLM から呼び出せるカスタムツールの登録、コマンド追加、状態永続化、UI 拡張などを行えます。ここでは導入方法からイベント、カスタムツール、UI、運用上の注意までを一通り説明します。

Extensions は、pi の挙動を拡張する TypeScript モジュールです。ライフサイクルイベントを購読したり、LLM から呼び出せるカスタムツールを登録したり、コマンドを追加したり、さらに多くのことができます。

> **`/reload` の配置場所:** 自動検出されるよう、拡張機能は `~/.pi/agent/extensions/`（グローバル）または `.pi/extensions/`（プロジェクトローカル）に置いてください。`pi -e ./path.ts` はクイックテスト用にのみ使ってください。自動検出場所にある拡張機能は `/reload` でホットリロードできます。

**主な機能:**
- **カスタムツール** - `pi.registerTool()` 経由で、LLM が呼び出せるツールを登録できます
- **イベント介入** - ツール呼び出しのブロックや変更、コンテキスト注入、コンパクションのカスタマイズができます
- **ユーザー操作** - `ctx.ui`（select、confirm、input、notify）を通じてユーザーに問い合わせできます
- **カスタム UI コンポーネント** - 複雑な対話向けに、`ctx.ui.custom()` でキーボード入力付きの完全な TUI コンポーネントを作れます
- **カスタムコマンド** - `pi.registerCommand()` で `/mycommand` のようなコマンドを登録できます
- **セッション永続化** - `pi.appendEntry()` で、再起動後も残る状態を保存できます
- **カスタムレンダリング** - ツール呼び出し/結果やメッセージの TUI 表示方法を制御できます

**利用例:**
- 権限ゲート（`rm -rf` や `sudo` の前に確認するなど）
- Git チェックポイント管理（各ターンで stash、ブランチで復元）
- パス保護（`.env` や `node_modules/` への書き込みをブロック）
- カスタムコンパクション（独自の会話要約）
- 会話要約（`summarize.ts` の例を参照）
- 対話型ツール（質問、ウィザード、カスタムダイアログ）
- 状態を持つツール（todo リスト、コネクションプール）
- 外部連携（ファイルウォッチャー、webhook、CI トリガー）
- 待ち時間中のゲーム（`snake.ts` の例を参照）

実装例は [examples/extensions/](../examples/extensions/) を参照してください。

## Table of Contents

- [Quick Start](#quick-start)
- [Extension Locations](#extension-locations)
- [Available Imports](#available-imports)
- [Writing an Extension](#writing-an-extension)
  - [Extension Styles](#extension-styles)
- [Events](#events)
  - [Lifecycle Overview](#lifecycle-overview)
  - [Resource Events](#resource-events)
  - [Session Events](#session-events)
  - [Agent Events](#agent-events)
  - [Model Events](#model-events)
  - [Tool Events](#tool-events)
- [ExtensionContext](#extensioncontext)
- [ExtensionCommandContext](#extensioncommandcontext)
- [ExtensionAPI Methods](#extensionapi-methods)
- [State Management](#state-management)
- [Custom Tools](#custom-tools)
- [Custom UI](#custom-ui)
- [Error Handling](#error-handling)
- [Mode Behavior](#mode-behavior)
- [Examples Reference](#examples-reference)

## Quick Start

`~/.pi/agent/extensions/my-extension.ts` を作成します:

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { Type } from "typebox";

export default function (pi: ExtensionAPI) {
  // React to events
  pi.on("session_start", async (_event, ctx) => {
    ctx.ui.notify("Extension loaded!", "info");
  });

  pi.on("tool_call", async (event, ctx) => {
    if (event.toolName === "bash" && event.input.command?.includes("rm -rf")) {
      const ok = await ctx.ui.confirm("Dangerous!", "Allow rm -rf?");
      if (!ok) return { block: true, reason: "Blocked by user" };
    }
  });

  // Register a custom tool
  pi.registerTool({
    name: "greet",
    label: "Greet",
    description: "Greet someone by name",
    parameters: Type.Object({
      name: Type.String({ description: "Name to greet" }),
    }),
    async execute(toolCallId, params, signal, onUpdate, ctx) {
      return {
        content: [{ type: "text", text: `Hello, ${params.name}!` }],
        details: {},
      };
    },
  });

  // Register a command
  pi.registerCommand("hello", {
    description: "Say hello",
    handler: async (args, ctx) => {
      ctx.ui.notify(`Hello ${args || "world"}!`, "info");
    },
  });
}
```

`--extension`（または `-e`）フラグでテストします:

```bash
pi -e ./my-extension.ts
```

## Extension Locations

> **セキュリティ:** 拡張機能はあなたのシステム権限を完全に引き継いで実行され、任意コードを実行できます。信頼できる提供元のものだけをインストールしてください。

Extensions は以下の場所から自動検出されます:

| Location | Scope |
|----------|-------|
| `~/.pi/agent/extensions/*.ts` | グローバル（すべてのプロジェクト） |
| `~/.pi/agent/extensions/*/index.ts` | グローバル（サブディレクトリ） |
| `.pi/extensions/*.ts` | プロジェクトローカル |
| `.pi/extensions/*/index.ts` | プロジェクトローカル（サブディレクトリ） |

`settings.json` で追加パスを指定することもできます:

```json
{
  "packages": [
    "npm:@foo/bar@1.0.0",
    "git:github.com/user/repo@v1"
  ],
  "extensions": [
    "/path/to/local/extension.ts",
    "/path/to/local/extension/dir"
  ]
}
```

npm や git 経由で pi パッケージとして拡張機能を共有する場合は、[packages.md](packages.md) を参照してください。

## Available Imports

| Package | 用途 |
|---------|---------|
| `@earendil-works/pi-coding-agent` | 拡張機能の型（`ExtensionAPI`、`ExtensionContext`、events） |
| `typebox` | ツール引数のスキーマ定義 |
| `@earendil-works/pi-ai` | AI ユーティリティ（Google 互換 enum 用の `StringEnum` など） |
| `@earendil-works/pi-tui` | カスタムレンダリング用の TUI コンポーネント |

npm 依存も利用できます。拡張機能の隣接ディレクトリ（または親ディレクトリ）に `package.json` を置いて `npm install` を実行すると、`node_modules/` からの import が自動解決されます。

`pi install`（npm または git）で導入される配布用 pi パッケージでは、ランタイム依存は `dependencies` に入っている必要があります。パッケージインストールは既定で本番用インストール（`npm install --omit=dev`）を使うため、`devDependencies` は実行時には利用できません。`npmCommand` が設定されている場合、git パッケージでは互換性のため通常の `install` が使われます。

Node.js 組み込みモジュール（`node:fs`、`node:path` など）も利用できます。

## Writing an Extension

拡張機能は、`ExtensionAPI` を受け取るデフォルトのファクトリ関数を export します。このファクトリは同期でも非同期でも構いません:

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";

export default function (pi: ExtensionAPI) {
  // Subscribe to events
  pi.on("event_name", async (event, ctx) => {
    // ctx.ui for user interaction
    const ok = await ctx.ui.confirm("Title", "Are you sure?");
    ctx.ui.notify("Done!", "info");
    ctx.ui.setStatus("my-ext", "Processing...");  // Footer status
    ctx.ui.setWidget("my-ext", ["Line 1", "Line 2"]);  // Widget above editor (default)
  });

  // Register tools, commands, shortcuts, flags
  pi.registerTool({ ... });
  pi.registerCommand("name", { ... });
  pi.registerShortcut("ctrl+x", { ... });
  pi.registerFlag("my-flag", { ... });
}
```

Extensions は [jiti](https://github.com/unjs/jiti) 経由で読み込まれるため、コンパイルなしで TypeScript を使えます。

ファクトリが `Promise` を返す場合、pi は起動継続前にその完了を待機します。つまり、非同期初期化は `session_start`、`resources_discover`、そして `pi.registerProvider()` によりキューされたプロバイダ登録が反映される前に完了します。

### Async factory functions

リモート設定の取得や、利用可能なモデルの動的検出のような一度きりの起動処理には、非同期ファクトリを使います。

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";

export default async function (pi: ExtensionAPI) {
  const response = await fetch("http://localhost:1234/v1/models");
  const payload = (await response.json()) as {
    data: Array<{
      id: string;
      name?: string;
      context_window?: number;
      max_tokens?: number;
    }>;
  };

  pi.registerProvider("local-openai", {
    baseUrl: "http://localhost:1234/v1",
    apiKey: "$LOCAL_OPENAI_API_KEY",
    api: "openai-completions",
    models: payload.data.map((model) => ({
      id: model.id,
      name: model.name ?? model.id,
      reasoning: false,
      input: ["text"],
      cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
      contextWindow: model.context_window ?? 128000,
      maxTokens: model.max_tokens ?? 4096,
    })),
  });
}
```

このパターンにより、取得したモデルは通常起動時にも `pi --list-models` に対しても利用可能になります。

### Extension Styles

**単一ファイル** - 小さな拡張機能向けの最も簡単な形:

```
~/.pi/agent/extensions/
└── my-extension.ts
```

**`index.ts` を持つディレクトリ** - 複数ファイルの拡張機能向け:

```
~/.pi/agent/extensions/
└── my-extension/
    ├── index.ts        # Entry point (exports default function)
    ├── tools.ts        # Helper module
    └── utils.ts        # Helper module
```

**依存を持つパッケージ** - npm パッケージが必要な拡張機能向け:

```
~/.pi/agent/extensions/
└── my-extension/
    ├── package.json    # Declares dependencies and entry points
    ├── package-lock.json
    ├── node_modules/   # After npm install
    └── src/
        └── index.ts
```

```json
// package.json
{
  "name": "my-extension",
  "dependencies": {
    "zod": "^3.0.0",
    "chalk": "^5.0.0"
  },
  "pi": {
    "extensions": ["./src/index.ts"]
  }
}
```

拡張機能ディレクトリで `npm install` を実行すると、`node_modules/` からの import が自動で使えるようになります。

## Events

### Lifecycle Overview

```
pi starts
  │
  ├─► session_start { reason: "startup" }
  └─► resources_discover { reason: "startup" }
      │
      ▼
user sends prompt ─────────────────────────────────────────┐
  │                                                        │
  ├─► (extension commands checked first, bypass if found)  │
  ├─► input (can intercept, transform, or handle)          │
  ├─► (skill/template expansion if not handled)            │
  ├─► before_agent_start (can inject message, modify system prompt)
  ├─► agent_start                                          │
  ├─► message_start / message_update / message_end         │
  │                                                        │
  │   ┌─── turn (repeats while LLM calls tools) ───┐       │
  │   │                                            │       │
  │   ├─► turn_start                               │       │
  │   ├─► context (can modify messages)            │       │
  │   ├─► before_provider_request (can inspect or replace payload)
  │   ├─► after_provider_response (status + headers, before stream consume)
  │   │                                            │       │
  │   │   LLM responds, may call tools:            │       │
  │   │     ├─► tool_execution_start               │       │
  │   │     ├─► tool_call (can block)              │       │
  │   │     ├─► tool_execution_update              │       │
  │   │     ├─► tool_result (can modify)           │       │
  │   │     └─► tool_execution_end                 │       │
  │   │                                            │       │
  │   └─► turn_end                                 │       │
  │                                                        │
  └─► agent_end                                            │
                                                           │
user sends another prompt ◄────────────────────────────────┘

/new (new session) or /resume (switch session)
  ├─► session_before_switch (can cancel)
  ├─► session_shutdown
  ├─► session_start { reason: "new" | "resume", previousSessionFile? }
  └─► resources_discover { reason: "startup" }

/fork or /clone
  ├─► session_before_fork (can cancel)
  ├─► session_shutdown
  ├─► session_start { reason: "fork", previousSessionFile }
  └─► resources_discover { reason: "startup" }

/compact or auto-compaction
  ├─► session_before_compact (can cancel or customize)
  └─► session_compact

/tree navigation
  ├─► session_before_tree (can cancel or customize)
  └─► session_tree

/model or Ctrl+P (model selection/cycling)
  ├─► thinking_level_select (if model change changes/clamps thinking level)
  └─► model_select

thinking level changes (settings, keybinding, pi.setThinkingLevel())
  └─► thinking_level_select

exit (Ctrl+C, Ctrl+D, SIGHUP, SIGTERM)
  └─► session_shutdown
```

### Resource Events

#### resources_discover

`session_start` の後に発火し、拡張機能が追加の skill、prompt、theme のパスを提供できるようにします。
起動時は `reason: "startup"`、リロード時は `reason: "reload"` になります。

```typescript
pi.on("resources_discover", async (event, _ctx) => {
  // event.cwd - current working directory
  // event.reason - "startup" | "reload"
  return {
    skillPaths: ["/path/to/skills"],
    promptPaths: ["/path/to/prompts"],
    themePaths: ["/path/to/themes"],
  };
});
```

### Session Events

セッション保存の内部仕様と SessionManager API については [Session Format](session-format.md) を参照してください。

#### session_start

セッションが開始、読み込み、または再読み込みされたときに発火します。

```typescript
pi.on("session_start", async (event, ctx) => {
  // event.reason - "startup" | "reload" | "new" | "resume" | "fork"
  // event.previousSessionFile - present for "new", "resume", and "fork"
  ctx.ui.notify(`Session: ${ctx.sessionManager.getSessionFile() ?? "ephemeral"}`, "info");
});
```

#### session_before_switch

新しいセッション開始（`/new`）またはセッション切り替え（`/resume`）の前に発火します。

```typescript
pi.on("session_before_switch", async (event, ctx) => {
  // event.reason - "new" or "resume"
  // event.targetSessionFile - session we're switching to (only for "resume")

  if (event.reason === "new") {
    const ok = await ctx.ui.confirm("Clear?", "Delete all messages?");
    if (!ok) return { cancel: true };
  }
});
```

切り替えまたは新規セッションの作成が成功すると、pi は古い拡張機能インスタンスに対して `session_shutdown` を発火し、新しいセッション用に拡張機能を再読み込み・再バインドした後、`reason: "new" | "resume"` と `previousSessionFile` を付けて `session_start` を発火します。
クリーンアップは `session_shutdown` で行い、その後 `session_start` でメモリ上の状態を再構築してください。

#### session_before_fork

`/fork` または `/clone` による分岐時に発火します。

```typescript
pi.on("session_before_fork", async (event, ctx) => {
  // event.entryId - ID of the selected entry
  // event.position - "before" for /fork, "at" for /clone
  return { cancel: true }; // Cancel fork/clone
  // OR
  return { skipConversationRestore: true }; // Reserved for future conversation restore control
});
```

fork または clone が成功すると、pi は古い拡張機能インスタンスに対して `session_shutdown` を発火し、新しいセッション用に拡張機能を再読み込み・再バインドした後、`reason: "fork"` と `previousSessionFile` を付けて `session_start` を発火します。
クリーンアップは `session_shutdown` で行い、その後 `session_start` でメモリ上の状態を再構築してください。

#### session_before_compact / session_compact

コンパクション時に発火します。詳細は [compaction.md](compaction.md) を参照してください。

```typescript
pi.on("session_before_compact", async (event, ctx) => {
  const { preparation, branchEntries, customInstructions, signal } = event;

  // Cancel:
  return { cancel: true };

  // Custom summary:
  return {
    compaction: {
      summary: "...",
      firstKeptEntryId: preparation.firstKeptEntryId,
      tokensBefore: preparation.tokensBefore,
    }
  };
});

pi.on("session_compact", async (event, ctx) => {
  // event.compactionEntry - the saved compaction
  // event.fromExtension - whether extension provided it
});
```

#### session_before_tree / session_tree

`/tree` ナビゲーション時に発火します。ツリー移動の概念は [Sessions](sessions.md) を参照してください。

```typescript
pi.on("session_before_tree", async (event, ctx) => {
  const { preparation, signal } = event;
  return { cancel: true };
  // OR provide custom summary:
  return { summary: { summary: "...", details: {} } };
});

pi.on("session_tree", async (event, ctx) => {
  // event.newLeafId, oldLeafId, summaryEntry, fromExtension
});
```

#### session_shutdown

拡張機能ランタイムが破棄される前に発火します。

```typescript
pi.on("session_shutdown", async (event, ctx) => {
  // event.reason - "quit" | "reload" | "new" | "resume" | "fork"
  // event.targetSessionFile - destination session for session replacement flows
  // Cleanup, save state, etc.
});
```

### Agent Events

#### before_agent_start

ユーザーがプロンプトを送信した後、エージェントループの前に発火します。メッセージ注入やシステムプロンプト変更が可能です。

```typescript
pi.on("before_agent_start", async (event, ctx) => {
  // event.prompt - user's prompt text
  // event.images - attached images (if any)
  // event.systemPrompt - current chained system prompt for this handler
  //   (includes changes from earlier before_agent_start handlers)
  // event.systemPromptOptions - structured options used to build the system prompt
  //   .customPrompt - any custom system prompt (from --system-prompt, SYSTEM.md, or custom templates)
  //   .selectedTools - tools currently active in the prompt
  //   .toolSnippets - one-line descriptions for each tool
  //   .promptGuidelines - custom guideline bullets
  //   .appendSystemPrompt - text from --append-system-prompt flags
  //   .cwd - working directory
  //   .contextFiles - AGENTS.md files and other loaded context files
  //   .skills - loaded skills

  return {
    // Inject a persistent message (stored in session, sent to LLM)
    message: {
      customType: "my-extension",
      content: "Additional context for the LLM",
      display: true,
    },
    // Replace the system prompt for this turn (chained across extensions)
    systemPrompt: event.systemPrompt + "\n\nExtra instructions for this turn...",
  };
});
```

`systemPromptOptions` フィールドにより、拡張機能は Pi がシステムプロンプトを組み立てるのに使っている構造化データへアクセスできます。これにより、リソースを再探索したりフラグを再解析したりせずに、Pi が何を読み込んだか（カスタムプロンプト、ガイドライン、ツール説明、コンテキストファイル、skills）を確認できます。ユーザー設定を尊重しながら、システムプロンプトへ深く踏み込んだ変更をしたい場合に使ってください。

`before_agent_start` 内では、`event.systemPrompt` と `ctx.getSystemPrompt()` はどちらも、そのハンドラ時点までにチェーンされたシステムプロンプトを反映します。後続の `before_agent_start` ハンドラがさらに変更する可能性はあります。

#### agent_start / agent_end

各ユーザープロンプトごとに 1 回発火します。

```typescript
pi.on("agent_start", async (_event, ctx) => {});

pi.on("agent_end", async (event, ctx) => {
  // event.messages - messages from this prompt
});
```

#### turn_start / turn_end

各ターン（1 回の LLM 応答 + ツール呼び出し）ごとに発火します。

```typescript
pi.on("turn_start", async (event, ctx) => {
  // event.turnIndex, event.timestamp
});

pi.on("turn_end", async (event, ctx) => {
  // event.turnIndex, event.message, event.toolResults
});
```

#### message_start / message_update / message_end

メッセージのライフサイクル更新時に発火します。

- `message_start` と `message_end` は user、assistant、toolResult メッセージに対して発火します
- `message_update` は assistant のストリーミング更新に対して発火します
- `message_end` ハンドラは `{ message }` を返して確定済みメッセージを置き換えられます。置き換え後も同じ `role` を維持する必要があります

```typescript
pi.on("message_start", async (event, ctx) => {
  // event.message
});

pi.on("message_update", async (event, ctx) => {
  // event.message
  // event.assistantMessageEvent (token-by-token stream event)
});

pi.on("message_end", async (event, ctx) => {
  if (event.message.role !== "assistant") return;

  return {
    message: {
      ...event.message,
      usage: {
        ...event.message.usage,
        cost: {
          ...event.message.usage.cost,
          total: 0.123,
        },
      },
    },
  };
});
```

#### tool_execution_start / tool_execution_update / tool_execution_end

ツール実行のライフサイクル更新時に発火します。

並列ツールモードでは:
- `tool_execution_start` は preflight フェーズ中に assistant のソース順で発火します
- `tool_execution_update` は複数ツール間で交互に発生することがあります
- `tool_execution_end` は各ツール確定後、完了順で発火します
- 最終的な `toolResult` メッセージイベントはその後も assistant のソース順で発火します

```typescript
pi.on("tool_execution_start", async (event, ctx) => {
  // event.toolCallId, event.toolName, event.args
});

pi.on("tool_execution_update", async (event, ctx) => {
  // event.toolCallId, event.toolName, event.args, event.partialResult
});

pi.on("tool_execution_end", async (event, ctx) => {
  // event.toolCallId, event.toolName, event.result, event.isError
});
```

#### context

各 LLM 呼び出し前に発火します。メッセージを破壊的でない形で変更できます。メッセージ型は [Session Format](session-format.md) を参照してください。

```typescript
pi.on("context", async (event, ctx) => {
  // event.messages - deep copy, safe to modify
  const filtered = event.messages.filter(m => !shouldPrune(m));
  return { messages: filtered };
});
```

#### before_provider_request

プロバイダ固有の payload が構築された直後、リクエスト送信直前に発火します。ハンドラは拡張機能の読み込み順に実行されます。`undefined` を返すと payload はそのまま維持されます。それ以外を返すと、後続ハンドラおよび実際のリクエストに対して payload が置き換えられます。

このフックでは、プロバイダレベルのシステム命令を書き換えたり完全に削除したりできます。こうした payload レベルの変更は `ctx.getSystemPrompt()` には反映されません。`ctx.getSystemPrompt()` が返すのは Pi のシステムプロンプト文字列であり、最終的にシリアライズされたプロバイダ payload ではないためです。

```typescript
pi.on("before_provider_request", (event, ctx) => {
  console.log(JSON.stringify(event.payload, null, 2));

  // Optional: replace payload
  // return { ...event.payload, temperature: 0 };
});
```

主な用途は、プロバイダへのシリアライズやキャッシュ挙動のデバッグです。

#### after_provider_response

HTTP レスポンス受信後、ストリーム本文を消費する前に発火します。ハンドラは拡張機能の読み込み順に実行されます。

```typescript
pi.on("after_provider_response", (event, ctx) => {
  // event.status - HTTP status code
  // event.headers - normalized response headers
  if (event.status === 429) {
    console.log("rate limited", event.headers["retry-after"]);
  }
});
```

ヘッダーの取得可否はプロバイダとトランスポートに依存します。HTTP レスポンスを抽象化しているプロバイダでは、ヘッダーが公開されない場合があります。

### Model Events

#### model_select

`/model` コマンド、モデル切り替え（`Ctrl+P`）、またはセッション復元によりモデルが変わったときに発火します。

```typescript
pi.on("model_select", async (event, ctx) => {
  // event.model - newly selected model
  // event.previousModel - previous model (undefined if first selection)
  // event.source - "set" | "cycle" | "restore"

  const prev = event.previousModel
    ? `${event.previousModel.provider}/${event.previousModel.id}`
    : "none";
  const next = `${event.model.provider}/${event.model.id}`;

  ctx.ui.notify(`Model changed (${event.source}): ${prev} -> ${next}`, "info");
});
```

アクティブモデル変更時に、UI 要素（ステータスバーやフッターなど）を更新したり、モデル固有の初期化処理を行ったりする用途に使います。

#### thinking_level_select

thinking level が変わったときに発火します。これは通知専用で、ハンドラの戻り値は無視されます。

```typescript
pi.on("thinking_level_select", async (event, ctx) => {
  // event.level - newly selected thinking level
  // event.previousLevel - previous thinking level

  ctx.ui.setStatus("thinking", `thinking: ${event.level}`);
});
```

`pi.setThinkingLevel()`、モデル変更、または組み込みの thinking-level 操作で active thinking level が変わるときに、拡張機能 UI を更新する用途に使います。

### Tool Events

#### tool_call

`tool_execution_start` の後、ツール実行前に発火します。**ブロック可能です。** `isToolCallEventType` を使うと型を絞り込み、型付き入力を扱えます。

`tool_call` 実行前に、pi は `AgentSession` を通じて先に発火済みの Agent event がすべて完了するまで待機します。つまり `ctx.sessionManager` は、現在の assistant のツール呼び出しメッセージまで反映済みです。

既定の並列ツール実行モードでは、同じ assistant メッセージ内の兄弟ツール呼び出しは順番に preflight された後、並行実行されます。そのため `tool_call` では、同じ assistant メッセージ中の兄弟ツール結果が `ctx.sessionManager` にまだ反映されていない場合があります。

`event.input` はミュータブルです。実行前にツール引数を書き換えるには、その場で変更してください。

挙動保証:
- `event.input` への変更は実際のツール実行に反映されます
- 後続の `tool_call` ハンドラは、先行ハンドラによる変更後の値を見ます
- 変更後の再バリデーションは行われません
- `tool_call` の戻り値が制御できるのは `{ block: true, reason?: string }` によるブロックだけです

```typescript
import { isToolCallEventType } from "@earendil-works/pi-coding-agent";

pi.on("tool_call", async (event, ctx) => {
  // event.toolName - "bash", "read", "write", "edit", etc.
  // event.toolCallId
  // event.input - tool parameters (mutable)

  // Built-in tools: no type params needed
  if (isToolCallEventType("bash", event)) {
    // event.input is { command: string; timeout?: number }
    event.input.command = `source ~/.profile\n${event.input.command}`;

    if (event.input.command.includes("rm -rf")) {
      return { block: true, reason: "Dangerous command" };
    }
  }

  if (isToolCallEventType("read", event)) {
    // event.input is { path: string; offset?: number; limit?: number }
    console.log(`Reading: ${event.input.path}`);
  }
});
```

#### Typing custom tool input

カスタムツールはその入力型を export するべきです:

```typescript
// my-extension.ts
export type MyToolInput = Static<typeof myToolSchema>;
```

`isToolCallEventType` を明示的な型引数付きで使います:

```typescript
import { isToolCallEventType } from "@earendil-works/pi-coding-agent";
import type { MyToolInput } from "my-extension";

pi.on("tool_call", (event) => {
  if (isToolCallEventType<"my_tool", MyToolInput>("my_tool", event)) {
    event.input.action;  // typed
  }
});
```

#### tool_result

ツール実行完了後、`tool_execution_end` と最終 `toolResult` メッセージイベントが発火する前に発火します。**結果を変更できます。**

並列ツールモードでは、`tool_result` と `tool_execution_end` はツール完了順で入り混じることがありますが、最終 `toolResult` メッセージイベントはその後も assistant のソース順で発火します。

`tool_result` ハンドラはミドルウェアのように連結されます:
- ハンドラは拡張機能の読み込み順に実行されます
- 各ハンドラは、先行ハンドラの変更が反映された最新の結果を見ます
- ハンドラは部分パッチ（`content`、`details`、`isError`）を返せます。省略したフィールドは現状維持されます

ハンドラ内でネストした非同期処理を行う場合は `ctx.signal` を使ってください。これにより、拡張機能が開始した model call、`fetch()`、その他 Abort 対応処理を Esc で中断できます。

```typescript
import { isBashToolResult } from "@earendil-works/pi-coding-agent";

pi.on("tool_result", async (event, ctx) => {
  // event.toolName, event.toolCallId, event.input
  // event.content, event.details, event.isError

  if (isBashToolResult(event)) {
    // event.details is typed as BashToolDetails
  }

  const response = await fetch("https://example.com/summarize", {
    method: "POST",
    body: JSON.stringify({ content: event.content }),
    signal: ctx.signal,
  });

  // Modify result:
  return { content: [...], details: {...}, isError: false };
});
```

### User Bash Events

#### user_bash

ユーザーが `!` または `!!` コマンドを実行したときに発火します。**介入可能です。**

```typescript
import { createLocalBashOperations } from "@earendil-works/pi-coding-agent";

pi.on("user_bash", (event, ctx) => {
  // event.command - the bash command
  // event.excludeFromContext - true if !! prefix
  // event.cwd - working directory

  // Option 1: Provide custom operations (e.g., SSH)
  return { operations: remoteBashOps };

  // Option 2: Wrap pi's built-in local bash backend
  const local = createLocalBashOperations();
  return {
    operations: {
      exec(command, cwd, options) {
        return local.exec(`source ~/.profile\n${command}`, cwd, options);
      }
    }
  };

  // Option 3: Full replacement - return result directly
  return { result: { output: "...", exitCode: 0, cancelled: false, truncated: false } };
});
```

### Input Events

#### input

ユーザー入力受信時、拡張コマンドのチェック後かつ skill/template 展開前に発火します。イベントは生の入力テキストを見るため、`/skill:foo` や `/template` はまだ展開されていません。

**処理順:**
1. 先に拡張コマンド（`/cmd`）を確認。見つかればハンドラが実行され、input event はスキップされます
2. `input` event が発火し、介入・変換・処理できます
3. 未処理なら skill コマンド（`/skill:name`）が skill 内容へ展開されます
4. 未処理なら prompt template（`/template`）がテンプレート内容へ展開されます
5. エージェント処理開始（`before_agent_start` など）

```typescript
pi.on("input", async (event, ctx) => {
  // event.text - raw input (before skill/template expansion)
  // event.images - attached images, if any
  // event.source - "interactive" (typed), "rpc" (API), or "extension" (via sendUserMessage)
  // event.streamingBehavior - "steer" | "followUp" | undefined
  //   undefined when idle, "steer" for mid-stream interrupts,
  //   "followUp" for messages queued until the agent finishes

  // Transform: rewrite input before expansion
  if (event.text.startsWith("?quick "))
    return { action: "transform", text: `Respond briefly: ${event.text.slice(7)}` };

  // Handle: respond without LLM (extension shows its own feedback)
  if (event.text === "ping") {
    ctx.ui.notify("pong", "info");
    return { action: "handled" };
  }

  // Route by source: skip processing for extension-injected messages
  if (event.source === "extension") return { action: "continue" };

  // Intercept skill commands before expansion
  if (event.text.startsWith("/skill:")) {
    // Could transform, block, or let pass through
  }

  return { action: "continue" };  // Default: pass through to expansion
});
```

**結果:**
- `continue` - 変更なしで通過（ハンドラが何も返さない場合の既定値）
- `transform` - text/images を変更してから展開へ進む
- `handled` - エージェント全体をスキップ（最初にこれを返したハンドラが勝ちます）

transform はハンドラ間で連鎖します。`streamingBehavior` を踏まえたルーティング例は [input-transform.ts](../examples/extensions/input-transform.ts) と [input-transform-streaming.ts](../examples/extensions/input-transform-streaming.ts) を参照してください。

## ExtensionContext

すべてのハンドラは `ctx: ExtensionContext` を受け取ります。

### ctx.ui

ユーザー操作用の UI メソッドです。詳細は [Custom UI](#custom-ui) を参照してください。

### ctx.hasUI

print mode（`-p`）および JSON mode では `false` です。interactive と RPC mode では `true` です。RPC mode では、ダイアログ系メソッド（`select`、`confirm`、`input`、`editor`）は extension UI sub-protocol 経由で動作し、fire-and-forget 系メソッド（`notify`、`setStatus`、`setWidget`、`setTitle`、`setEditorText`）はクライアントへのリクエストを発行します。一部の TUI 専用メソッドは no-op になったり既定値を返したりします（[rpc.md](rpc.md#extension-ui-protocol) を参照）。

### ctx.cwd

現在の作業ディレクトリです。

### ctx.sessionManager

セッション状態への読み取り専用アクセスです。完全な SessionManager API とエントリ型は [Session Format](session-format.md) を参照してください。

`tool_call` では、ハンドラ実行前にこの状態が現在の assistant message まで同期されています。ただし並列ツール実行モードでは、同じ assistant message の兄弟ツール結果が含まれる保証はありません。

```typescript
ctx.sessionManager.getEntries()       // All entries
ctx.sessionManager.getBranch()        // Current branch
ctx.sessionManager.getLeafId()        // Current leaf entry ID
```

### ctx.modelRegistry / ctx.model

モデルと API キーへのアクセスです。

### ctx.signal

現在の agent abort signal、または active な agent turn がなければ `undefined` です。

拡張機能ハンドラが開始する Abort 対応のネスト処理に使ってください。例:
- `fetch(..., { signal: ctx.signal })`
- `signal` を受け取る model call
- `AbortSignal` を受け取るファイル/プロセス helper

`ctx.signal` は通常、`tool_call`、`tool_result`、`message_update`、`turn_end` など active turn 中の event で定義されます。
`session` event、拡張コマンド、pi が idle なときの shortcut など、非ターン文脈では通常 `undefined` です。

```typescript
pi.on("tool_result", async (event, ctx) => {
  const response = await fetch("https://example.com/api", {
    method: "POST",
    body: JSON.stringify(event),
    signal: ctx.signal,
  });

  const data = await response.json();
  return { details: data };
});
```

### ctx.isIdle() / ctx.abort() / ctx.hasPendingMessages()

制御フロー用の helper です。

### ctx.shutdown()

pi の正常終了を要求します。

- **Interactive mode:** agent が idle になるまで遅延されます（キュー済みの steering / follow-up message をすべて処理した後）
- **RPC mode:** 次に idle になったタイミングまで遅延されます（現在のコマンド応答完了後、次のコマンド待ちに入るとき）
- **Print mode:** no-op です。すべての prompt を処理し終えると自動終了します

終了前に、すべての extension に対して `session_shutdown` event が発火します。event handler、tool、command、shortcut を問わず全コンテキストで利用できます。

```typescript
pi.on("tool_call", (event, ctx) => {
  if (isFatal(event.input)) {
    ctx.shutdown();
  }
});
```

### ctx.getContextUsage()

アクティブモデルに対する現在の context usage を返します。可能なら直近 assistant usage を使い、それ以外では末尾メッセージの token 数を推定します。

```typescript
const usage = ctx.getContextUsage();
if (usage && usage.tokens > 100_000) {
  // ...
}
```

### ctx.compact()

完了を待たずに compaction をトリガーします。後続処理には `onComplete` と `onError` を使います。

```typescript
ctx.compact({
  customInstructions: "Focus on recent changes",
  onComplete: (result) => {
    ctx.ui.notify("Compaction completed", "info");
  },
  onError: (error) => {
    ctx.ui.notify(`Compaction failed: ${error.message}`, "error");
  },
});
```

### ctx.getSystemPrompt()

Pi の現在の system prompt 文字列を返します。

- `before_agent_start` 中では、その時点までにチェーンされた system prompt 変更を反映します
- 後続の `context` message 変更は含みません
- `before_provider_request` の payload 書き換えも含みません
- あなたの後に読み込まれた拡張機能が、最終的に送信される内容をさらに変更することがあります

```typescript
pi.on("before_agent_start", (event, ctx) => {
  const prompt = ctx.getSystemPrompt();
  console.log(`System prompt length: ${prompt.length}`);
});
```

## ExtensionCommandContext

コマンドハンドラは `ExtensionCommandContext` を受け取ります。これは `ExtensionContext` を拡張し、セッション制御メソッドを追加したものです。これらは event handler から呼ぶとデッドロックしうるため、コマンド内でのみ利用できます。

### ctx.waitForIdle()

agent のストリーミング完了を待ちます:

```typescript
pi.registerCommand("my-cmd", {
  handler: async (args, ctx) => {
    await ctx.waitForIdle();
    // Agent is now idle, safe to modify session
  },
});
```

### ctx.newSession(options?)

新しいセッションを作成します:

```typescript
const parentSession = ctx.sessionManager.getSessionFile();
const kickoff = "Continue in the replacement session";

const result = await ctx.newSession({
  parentSession,
  setup: async (sm) => {
    sm.appendMessage({
      role: "user",
      content: [{ type: "text", text: "Context from previous session..." }],
      timestamp: Date.now(),
    });
  },
  withSession: async (ctx) => {
    // Use only the replacement-session ctx here.
    await ctx.sendUserMessage(kickoff);
  },
});

if (result.cancelled) {
  // An extension cancelled the new session
}
```

オプション:
- `parentSession`: 新しいセッションヘッダに記録する親セッションファイル
- `setup`: `withSession` 実行前に新規セッションの `SessionManager` を変更する
- `withSession`: 新しい置換後セッションの context に対して後処理を実行する。キャプチャした古い `pi` / command `ctx` は使わないでください。詳細は [Session replacement lifecycle and footguns](#session-replacement-lifecycle-and-footguns) を参照してください

### ctx.fork(entryId, options?)

特定のエントリから fork し、新しいセッションファイルを作成します:

```typescript
const result = await ctx.fork("entry-id-123", {
  withSession: async (ctx) => {
    // Use only the replacement-session ctx here.
    ctx.ui.notify("Now in the forked session", "info");
  },
});
if (result.cancelled) {
  // An extension cancelled the fork
}

const cloneResult = await ctx.fork("entry-id-456", { position: "at" });
if (cloneResult.cancelled) {
  // An extension cancelled the clone
}
```

オプション:
- `position`: `"before"`（既定値）では選択した user message の前で fork し、その prompt を editor に復元します
- `position`: `"at"` では editor text を復元せず、選択したエントリまでの active path を複製します
- `withSession`: 新しい置換後セッションの context に対して後処理を実行する。キャプチャした古い `pi` / command `ctx` は使わないでください。詳細は [Session replacement lifecycle and footguns](#session-replacement-lifecycle-and-footguns) を参照してください

### ctx.navigateTree(targetId, options?)

セッションツリー内の別地点へ移動します:

```typescript
const result = await ctx.navigateTree("entry-id-456", {
  summarize: true,
  customInstructions: "Focus on error handling changes",
  replaceInstructions: false, // true = replace default prompt entirely
  label: "review-checkpoint",
});
```

オプション:
- `summarize`: 破棄するブランチの要約を生成するかどうか
- `customInstructions`: 要約器向けのカスタム指示
- `replaceInstructions`: `true` の場合、`customInstructions` は既定プロンプトへ追記されず、完全置換されます
- `label`: ブランチ要約エントリ（要約しない場合は対象エントリ）へ付与するラベル

### ctx.switchSession(sessionPath, options?)

別のセッションファイルへ切り替えます:

```typescript
const result = await ctx.switchSession("/path/to/session.jsonl", {
  withSession: async (ctx) => {
    await ctx.sendUserMessage("Resume work in the replacement session");
  },
});
if (result.cancelled) {
  // An extension cancelled the switch via session_before_switch
}
```

オプション:
- `withSession`: 新しい置換後セッションの context に対して後処理を実行する。キャプチャした古い `pi` / command `ctx` は使わないでください。詳細は [Session replacement lifecycle and footguns](#session-replacement-lifecycle-and-footguns) を参照してください

利用可能なセッション一覧を取得するには、`SessionManager.list()` または `SessionManager.listAll()` の static method を使います:

```typescript
import { SessionManager } from "@earendil-works/pi-coding-agent";

pi.registerCommand("switch", {
  description: "Switch to another session",
  handler: async (args, ctx) => {
    const sessions = await SessionManager.list(ctx.cwd);
    if (sessions.length === 0) return;
    const choice = await ctx.ui.select(
      "Pick session:",
      sessions.map(s => s.file),
    );
    if (choice) {
      await ctx.switchSession(choice, {
        withSession: async (ctx) => {
          ctx.ui.notify("Switched session", "info");
        },
      });
    }
  },
});
```

### Session replacement lifecycle and footguns

`withSession` は、新しい `ReplacedSessionContext` を受け取ります。これは `ExtensionCommandContext` を拡張し、置換後セッションに束縛された非同期 `sendMessage()` / `sendUserMessage()` helper を提供します。

ライフサイクルと注意点:
- `withSession` は、古いセッションが `session_shutdown` を発火し、古いランタイムが破棄され、置換後セッションが再バインドされ、新しい拡張機能インスタンスがすでに `session_start` を受け取った後にのみ実行されます
- この callback 自体は新しい拡張機能インスタンスの中ではなく、元のクロージャ上で実行されます。つまり `withSession` 開始前に、古い拡張機能インスタンスがすでに shutdown cleanup を済ませている可能性があります
- キャプチャした古い `pi` / 古い command `ctx` のようなセッション束縛オブジェクトは、置換後は stale であり、使うと例外になります。セッション束縛処理には `withSession` に渡された `ctx` だけを使ってください
- 事前に取り出しておいた生オブジェクトの扱いは引き続き利用者責任です。たとえば置換前に `const sm = ctx.sessionManager` としていた場合、その `sm` は古い `SessionManager` オブジェクトのままです。置換後に再利用してはいけません
- `withSession` 内のコードは、`session_shutdown` handler により無効化される状態がすでに失われている前提で書くべきです。shutdown 後も安全に残る plain data（文字列、id、シリアライズ済み設定など）だけをキャプチャしてください

安全なパターン:

```typescript
pi.registerCommand("handoff", {
  handler: async (_args, ctx) => {
    const kickoff = "Continue from the replacement session";
    await ctx.newSession({
      withSession: async (ctx) => {
        await ctx.sendUserMessage(kickoff);
      },
    });
  },
});
```

危険なパターン:

```typescript
pi.registerCommand("handoff", {
  handler: async (_args, ctx) => {
    const oldSessionManager = ctx.sessionManager;
    await ctx.newSession({
      withSession: async (_ctx) => {
        // stale old objects: do not do this
        oldSessionManager.getSessionFile();
        pi.sendUserMessage("wrong");
      },
    });
  },
});
```

### ctx.reload()

`/reload` と同じリロードフローを実行します。

```typescript
pi.registerCommand("reload-runtime", {
  description: "Reload extensions, skills, prompts, and themes",
  handler: async (_args, ctx) => {
    await ctx.reload();
    return;
  },
});
```

重要な挙動:
- `await ctx.reload()` は、現在の拡張機能ランタイムへ `session_shutdown` を発火します
- 続いてリソースを再読み込みし、`reason: "reload"` 付き `session_start` と `reason: "reload"` 付き `resources_discover` を発火します
- 現在実行中の command handler 自体は、古い call frame 上で継続します
- `await ctx.reload()` の後のコードは、リロード前バージョンのまま動き続けます
- `await ctx.reload()` の後では、古いメモリ上の拡張状態がまだ有効だと仮定してはいけません
- handler が return した後、以後の command/event/tool call は新しい拡張バージョンを使います

予測しやすい挙動にするには、その handler では reload を終端扱いにしてください（`await ctx.reload(); return;`）。

tool は `ExtensionContext` で動作するため、`ctx.reload()` を直接呼べません。reload への入口は command にし、その command を follow-up user message としてキューする tool を公開してください。

LLM が呼べる reload トリガー用 tool の例:

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { Type } from "typebox";

export default function (pi: ExtensionAPI) {
  pi.registerCommand("reload-runtime", {
    description: "Reload extensions, skills, prompts, and themes",
    handler: async (_args, ctx) => {
      await ctx.reload();
      return;
    },
  });

  pi.registerTool({
    name: "reload_runtime",
    label: "Reload Runtime",
    description: "Reload extensions, skills, prompts, and themes",
    parameters: Type.Object({}),
    async execute() {
      pi.sendUserMessage("/reload-runtime", { deliverAs: "followUp" });
      return {
        content: [{ type: "text", text: "Queued /reload-runtime as a follow-up command." }],
      };
    },
  });
}
```

## ExtensionAPI Methods

### pi.on(event, handler)

イベントを購読します。event type と戻り値は [Events](#events) を参照してください。

### pi.registerTool(definition)

LLM が呼べるカスタムツールを登録します。詳細は [Custom Tools](#custom-tools) を参照してください。

`pi.registerTool()` は拡張機能読み込み時だけでなく、起動後にも使えます。`session_start`、command handler、その他 event handler の中から呼び出せます。新しい tool は同一セッション内ですぐ反映されるため、`pi.getAllTools()` に現れ、`/reload` なしで LLM から呼び出せます。

動的に追加した tool も含め、runtime 中に tool の有効/無効を切り替えるには `pi.setActiveTools()` を使ってください。

`promptSnippet` を指定すると、そのカスタム tool を `Available tools` セクションの 1 行説明へ参加させられます。`promptGuidelines` は、その tool が active な間だけ、既定の `Guidelines` セクションへ tool 固有の bullet を追記します。

**重要:** `promptGuidelines` の bullet は、tool 名プレフィックスなしで `Guidelines` セクションへフラットに追記されます。各 guideline は、どの tool を指しているかを明示する必要があります。"Use this tool when..." のように書くと、LLM は "this" がどの tool か判別できません。代わりに "Use my_tool when..." のように書いてください。

完全な例は [dynamic-tools.ts](../examples/extensions/dynamic-tools.ts) を参照してください。

```typescript
import { Type } from "typebox";
import { StringEnum } from "@earendil-works/pi-ai";

pi.registerTool({
  name: "my_tool",
  label: "My Tool",
  description: "What this tool does",
  promptSnippet: "Summarize or transform text according to action",
  promptGuidelines: ["Use my_tool when the user asks to summarize previously generated text."],
  parameters: Type.Object({
    action: StringEnum(["list", "add"] as const),
    text: Type.Optional(Type.String()),
  }),
  prepareArguments(args) {
    // Optional compatibility shim. Runs before schema validation.
    // Return the current schema shape, for example to fold legacy fields
    // into the modern parameter object.
    return args;
  },

  async execute(toolCallId, params, signal, onUpdate, ctx) {
    // Stream progress
    onUpdate?.({ content: [{ type: "text", text: "Working..." }] });

    return {
      content: [{ type: "text", text: "Done" }],
      details: { result: "..." },
    };
  },

  // Optional: Custom rendering
  renderCall(args, theme, context) { ... },
  renderResult(result, options, theme, context) { ... },
});
```

### pi.sendMessage(message, options?)

カスタムメッセージをセッションへ注入します。

```typescript
pi.sendMessage({
  customType: "my-extension",
  content: "Message text",
  display: true,
  details: { ... },
}, {
  triggerTurn: true,
  deliverAs: "steer",
});
```

**オプション:**
- `deliverAs` - 配送モード:
  - `"steer"`（既定値）- ストリーミング中にメッセージをキューします。現在の assistant turn が tool 呼び出しを終えたあと、次の LLM call の前に配送されます
  - `"followUp"` - agent が完了するまで待ちます。agent にこれ以上 tool call がなくなったときだけ配送されます
  - `"nextTurn"` - 次の user prompt 用にキューされます。割り込みもトリガーもしません
- `triggerTurn: true` - agent が idle なら即座に LLM 応答を開始します。`"steer"` と `"followUp"` にのみ適用されます（`"nextTurn"` では無視されます）

### pi.sendUserMessage(content, options?)

agent に user message を送信します。カスタムメッセージを送る `sendMessage()` と違い、これはユーザーが入力したかのように見える実際の user message を送ります。常に turn を開始します。

```typescript
// Simple text message
pi.sendUserMessage("What is 2+2?");

// With content array (text + images)
pi.sendUserMessage([
  { type: "text", text: "Describe this image:" },
  { type: "image", source: { type: "base64", mediaType: "image/png", data: "..." } },
]);

// During streaming - must specify delivery mode
pi.sendUserMessage("Focus on error handling", { deliverAs: "steer" });
pi.sendUserMessage("And then summarize", { deliverAs: "followUp" });
```

**オプション:**
- `deliverAs` - agent がストリーミング中は必須:
  - `"steer"` - 現在の assistant turn が tool 呼び出しを終えたあとで配送するようメッセージをキューします
  - `"followUp"` - agent がすべての tool を終えるまで待ちます

ストリーミング中でなければ、メッセージは即時送信されて新しい turn を開始します。ストリーミング中に `deliverAs` なしで呼ぶとエラーになります。

完全な例は [send-user-message.ts](../examples/extensions/send-user-message.ts) を参照してください。

### pi.appendEntry(customType, data?)

拡張機能の状態を永続化します（LLM context には参加しません）。

```typescript
pi.appendEntry("my-state", { count: 42 });

// Restore on reload
pi.on("session_start", async (_event, ctx) => {
  for (const entry of ctx.sessionManager.getEntries()) {
    if (entry.type === "custom" && entry.customType === "my-state") {
      // Reconstruct from entry.data
    }
  }
});
```

### pi.setSessionName(name)

セッションの表示名を設定します（最初のメッセージの代わりにセッションセレクタへ表示されます）。

```typescript
pi.setSessionName("Refactor auth module");
```

### pi.getSessionName()

設定されていれば、現在のセッション名を取得します。

```typescript
const name = pi.getSessionName();
if (name) {
  console.log(`Session: ${name}`);
}
```

### pi.setLabel(entryId, label)

エントリにラベルを設定または削除します。ラベルはブックマークやナビゲーション用のユーザー定義マーカーで、`/tree` セレクタに表示されます。

```typescript
// Set a label
pi.setLabel(entryId, "checkpoint-before-refactor");

// Clear a label
pi.setLabel(entryId, undefined);

// Read labels via sessionManager
const label = ctx.sessionManager.getLabel(entryId);
```

ラベルはセッションに永続化され、再起動後も残ります。会話ツリー内の重要な地点（turn、checkpoint など）を示すのに使ってください。

### pi.registerCommand(name, options)

コマンドを登録します。

複数の拡張機能が同じコマンド名を登録した場合、pi はそれらをすべて保持し、読み込み順で `/review:1`、`/review:2` のような数値 suffix を付けて呼び出せるようにします。

```typescript
pi.registerCommand("stats", {
  description: "Show session statistics",
  handler: async (args, ctx) => {
    const count = ctx.sessionManager.getEntries().length;
    ctx.ui.notify(`${count} entries`, "info");
  }
});
```

オプションとして、`/command ...` 用の引数オートコンプリートも追加できます:

```typescript
import type { AutocompleteItem } from "@earendil-works/pi-tui";

pi.registerCommand("deploy", {
  description: "Deploy to an environment",
  getArgumentCompletions: (prefix: string): AutocompleteItem[] | null => {
    const envs = ["dev", "staging", "prod"];
    const items = envs.map((e) => ({ value: e, label: e }));
    const filtered = items.filter((i) => i.value.startsWith(prefix));
    return filtered.length > 0 ? filtered : null;
  },
  handler: async (args, ctx) => {
    ctx.ui.notify(`Deploying: ${args}`, "info");
  },
});
```

### pi.getCommands()

現在のセッションで `prompt` 経由で呼び出せる slash command の一覧を取得します。extension command、prompt template、skill command を含みます。
この一覧の順序は RPC `get_commands` と一致し、extensions が先、その後 templates、最後に skills が続きます。

```typescript
const commands = pi.getCommands();
const bySource = commands.filter((command) => command.source === "extension");
const userScoped = commands.filter((command) => command.sourceInfo.scope === "user");
```

各要素の形は次のとおりです:

```typescript
{
  name: string; // Invokable command name without the leading slash. May be suffixed like "review:1"
  description?: string;
  source: "extension" | "prompt" | "skill";
  sourceInfo: {
    path: string;
    source: string;
    scope: "user" | "project" | "temporary";
    origin: "package" | "top-level";
    baseDir?: string;
  };
}
```

正規の provenance 情報としては `sourceInfo` を使ってください。コマンド名や場当たり的な path 解析から所有元を推測してはいけません。

`/model` や `/settings` のような組み込み interactive command はここには含まれません。これらは interactive mode でのみ処理され、`prompt` 経由で送っても実行されないためです。

### pi.registerMessageRenderer(customType, renderer)

あなたの `customType` を持つメッセージ用にカスタム TUI renderer を登録します。詳細は [Custom UI](#custom-ui) を参照してください。

### pi.registerShortcut(shortcut, options)

キーボードショートカットを登録します。shortcut 形式と組み込みキーバインドは [keybindings.md](keybindings.md) を参照してください。

```typescript
pi.registerShortcut("ctrl+shift+p", {
  description: "Toggle plan mode",
  handler: async (ctx) => {
    ctx.ui.notify("Toggled!");
  },
});
```

### pi.registerFlag(name, options)

CLI フラグを登録します。

```typescript
pi.registerFlag("plan", {
  description: "Start in plan mode",
  type: "boolean",
  default: false,
});

// Check value
if (pi.getFlag("plan")) {
  // Plan mode enabled
}
```

### pi.exec(command, args, options?)

shell command を実行します。

```typescript
const result = await pi.exec("git", ["status"], { signal, timeout: 5000 });
// result.stdout, result.stderr, result.code, result.killed
```

### pi.getActiveTools() / pi.getAllTools() / pi.setActiveTools(names)

active tool を管理します。これは組み込み tool と動的登録 tool の両方に使えます。

```typescript
const active = pi.getActiveTools();
const all = pi.getAllTools();
// [{
//   name: "read",
//   description: "Read file contents...",
//   parameters: ...,
//   promptGuidelines: ["Use read to examine files instead of cat or sed."],
//   sourceInfo: { path: "<builtin:read>", source: "builtin", scope: "temporary", origin: "top-level" }
// }, ...]
const names = all.map(t => t.name);
const builtinTools = all.filter((t) => t.sourceInfo.source === "builtin");
const extensionTools = all.filter((t) => t.sourceInfo.source !== "builtin" && t.sourceInfo.source !== "sdk");
pi.setActiveTools(["read", "bash"]); // Switch to read-only
```

`pi.getAllTools()` は `name`、`description`、`parameters`、`promptGuidelines`、`sourceInfo` を返します。

典型的な `sourceInfo.source` の値:
- `builtin` は組み込み tool
- `sdk` は `createAgentSession({ customTools })` 経由で渡された tool
- それ以外は extension で登録された tool の source metadata

### pi.setModel(model)

現在のモデルを設定します。モデルに利用可能な API キーがない場合は `false` を返します。カスタムモデル設定は [models.md](models.md) を参照してください。

```typescript
const model = ctx.modelRegistry.find("anthropic", "claude-sonnet-4-5");
if (model) {
  const success = await pi.setModel(model);
  if (!success) {
    ctx.ui.notify("No API key for this model", "error");
  }
}
```

### pi.getThinkingLevel() / pi.setThinkingLevel(level)

thinking level の取得と設定を行います。level はモデル能力に合わせて clamp されます（reasoning 非対応モデルは常に `"off"` です）。変更時には `thinking_level_select` が発火します。

```typescript
const current = pi.getThinkingLevel();  // "off" | "minimal" | "low" | "medium" | "high" | "xhigh"
pi.setThinkingLevel("high");
```

### pi.events

extension 間通信用の共有 event bus です:

```typescript
pi.events.on("my:event", (data) => { ... });
pi.events.emit("my:event", { ... });
```

### pi.registerProvider(name, config)

モデル provider を動的に登録または上書きします。proxy、カスタム endpoint、チーム共通のモデル設定などに有用です。

extension factory 中に行われた呼び出しはキューされ、runner 初期化時に適用されます。それ以外、たとえばユーザー設定フロー後の command handler から行った呼び出しは、`/reload` なしで即時反映されます。

リモート endpoint からモデルを検出したい場合は、`session_start` まで fetch を遅らせるのではなく、非同期 extension factory を使ってください。pi は起動継続前に factory 完了を待つため、登録モデルは `pi --list-models` を含め即時利用可能になります。

```typescript
// Register a new provider with custom models
pi.registerProvider("my-proxy", {
  name: "My Proxy",
  baseUrl: "https://proxy.example.com",
  apiKey: "$PROXY_API_KEY",  // env var reference
  api: "anthropic-messages",
  models: [
    {
      id: "claude-sonnet-4-20250514",
      name: "Claude 4 Sonnet (proxy)",
      reasoning: false,
      input: ["text", "image"],
      cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
      contextWindow: 200000,
      maxTokens: 16384
    }
  ]
});

// Override baseUrl for an existing provider (keeps all models)
pi.registerProvider("anthropic", {
  baseUrl: "https://proxy.example.com"
});

// Register provider with OAuth support for /login
pi.registerProvider("corporate-ai", {
  baseUrl: "https://ai.corp.com",
  api: "openai-responses",
  models: [...],
  oauth: {
    name: "Corporate AI (SSO)",
    async login(callbacks) {
      // Custom OAuth flow
      callbacks.onAuth({ url: "https://sso.corp.com/..." });
      const code = await callbacks.onPrompt({ message: "Enter code:" });
      return { refresh: code, access: code, expires: Date.now() + 3600000 };
    },
    async refreshToken(credentials) {
      // Refresh logic
      return credentials;
    },
    getApiKey(credentials) {
      return credentials.access;
    }
  }
});
```

**設定オプション:**
- `name` - `/login` などの UI に表示する provider 名
- `baseUrl` - API endpoint URL。model を定義する場合は必須
- `apiKey` - API key のリテラル、環境変数展開（`$ENV_VAR` または `${ENV_VAR}`）、または先頭 `!command`。model 定義時は必須（`oauth` がある場合を除く）。`$$` は `$` をエスケープし、`$!` は command 実行を発生させずにリテラル `!` を表します
- `api` - API type: `"anthropic-messages"`、`"openai-completions"`、`"openai-responses"` など
- `headers` - リクエストへ含めるカスタムヘッダー
- `authHeader` - `true` の場合、自動で `Authorization: Bearer` ヘッダーを追加
- `models` - model 定義の配列。指定すると、この provider の既存 model はすべて置き換えられます。各 model 定義は `baseUrl` を持てるため、その model に限って provider endpoint を上書きできます
- `oauth` - `/login` 対応用の OAuth provider 設定。指定すると、その provider が login menu に現れます
- `streamSimple` - 非標準 API 用のカスタム streaming 実装

カスタム streaming API、OAuth 詳細、model 定義の仕様など高度な話題は [custom-provider.md](custom-provider.md) を参照してください。

### pi.unregisterProvider(name)

以前に登録した provider とその models を削除します。provider により上書きされていた組み込み model は復元されます。provider が登録されていなければ何もしません。

`registerProvider` と同様に、初期読み込み後に呼べば即時反映されるため、`/reload` は不要です。

```typescript
pi.registerCommand("my-setup-teardown", {
  description: "Remove the custom proxy provider",
  handler: async (_args, _ctx) => {
    pi.unregisterProvider("my-proxy");
  },
});
```
