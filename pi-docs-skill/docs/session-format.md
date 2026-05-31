# セッションファイル形式

> 要約: Pi のセッションは JSONL で保存され、`id` と `parentId` によるツリー構造で分岐を表現します。このページはエントリ型、メッセージ型、文脈構築の流れ、`SessionManager` API まで含めてセッション内部表現を整理したものです。

セッションは JSONL（JSON Lines）ファイルとして保存されます。各行は `type` フィールドを持つ JSON オブジェクトです。セッションエントリは `id` / `parentId` フィールドでツリー構造を形成し、新しいファイルを作らずにその場で分岐できます。

## ファイルの場所

```
~/.pi/agent/sessions/--<path>--/<timestamp>_<uuid>.jsonl
```

ここで `<path>` は作業ディレクトリ中の `/` を `-` に置き換えたものです。

## セッションの削除

セッションは `~/.pi/agent/sessions/` 配下の `.jsonl` ファイルを削除することで除去できます。

Pi は `/resume` からの対話的削除もサポートしています（セッションを選択して `Ctrl+D` を押し、その後確認します）。利用可能な場合、pi は完全削除を避けるために `trash` CLI を使います。

## セッションバージョン

セッションのヘッダーにはバージョンフィールドがあります。

- **Version 1**: 線形のエントリ列（レガシー、読み込み時に自動移行）
- **Version 2**: `id` / `parentId` によるリンクを持つツリー構造
- **Version 3**: `hookMessage` ロールを `custom` に改名（拡張機能の統一）

既存セッションは、読み込み時に現在のバージョン（v3）へ自動的に移行されます。

## ソースファイル

GitHub 上のソース（[pi-mono](https://github.com/earendil-works/pi-mono)）:
- [`packages/coding-agent/src/core/session-manager.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/session-manager.ts) - セッションエントリ型と SessionManager
- [`packages/coding-agent/src/core/messages.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/messages.ts) - 拡張メッセージ型（BashExecutionMessage, CustomMessage など）
- [`packages/ai/src/types.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/ai/src/types.ts) - 基本メッセージ型（UserMessage, AssistantMessage, ToolResultMessage）
- [`packages/agent/src/types.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/agent/src/types.ts) - `AgentMessage` の union 型

プロジェクト内の TypeScript 定義は、`node_modules/@earendil-works/pi-coding-agent/dist/` と `node_modules/@earendil-works/pi-ai/dist/` を確認してください。

## メッセージ型

セッションエントリは `AgentMessage` オブジェクトを含みます。これらの型を理解することは、セッションのパースや拡張機能の実装に不可欠です。

### コンテンツブロック

メッセージは型付きコンテンツブロックの配列を含みます。

```typescript
interface TextContent {
  type: "text";
  text: string;
}

interface ImageContent {
  type: "image";
  data: string;      // base64 encoded
  mimeType: string;  // e.g., "image/jpeg", "image/png"
}

interface ThinkingContent {
  type: "thinking";
  thinking: string;
}

interface ToolCall {
  type: "toolCall";
  id: string;
  name: string;
  arguments: Record<string, any>;
}
```

### 基本メッセージ型（pi-ai 由来）

```typescript
interface UserMessage {
  role: "user";
  content: string | (TextContent | ImageContent)[];
  timestamp: number;  // Unix ms
}

interface AssistantMessage {
  role: "assistant";
  content: (TextContent | ThinkingContent | ToolCall)[];
  api: string;
  provider: string;
  model: string;
  usage: Usage;
  stopReason: "stop" | "length" | "toolUse" | "error" | "aborted";
  errorMessage?: string;
  timestamp: number;
}

interface ToolResultMessage {
  role: "toolResult";
  toolCallId: string;
  toolName: string;
  content: (TextContent | ImageContent)[];
  details?: any;      // Tool-specific metadata
  isError: boolean;
  timestamp: number;
}

interface Usage {
  input: number;
  output: number;
  cacheRead: number;
  cacheWrite: number;
  totalTokens: number;
  cost: {
    input: number;
    output: number;
    cacheRead: number;
    cacheWrite: number;
    total: number;
  };
}
```

### 拡張メッセージ型（pi-coding-agent 由来）

```typescript
interface BashExecutionMessage {
  role: "bashExecution";
  command: string;
  output: string;
  exitCode: number | undefined;
  cancelled: boolean;
  truncated: boolean;
  fullOutputPath?: string;
  excludeFromContext?: boolean;  // !! 接頭辞のコマンドでは true
  timestamp: number;
}

interface CustomMessage {
  role: "custom";
  customType: string;            // 拡張機能識別子
  content: string | (TextContent | ImageContent)[];
  display: boolean;              // TUI に表示するか
  details?: any;                 // 拡張機能固有のメタデータ
  timestamp: number;
}

interface BranchSummaryMessage {
  role: "branchSummary";
  summary: string;
  fromId: string;                // 分岐元のエントリ
  timestamp: number;
}

interface CompactionSummaryMessage {
  role: "compactionSummary";
  summary: string;
  tokensBefore: number;
  timestamp: number;
}
```

### AgentMessage の union

```typescript
type AgentMessage =
  | UserMessage
  | AssistantMessage
  | ToolResultMessage
  | BashExecutionMessage
  | CustomMessage
  | BranchSummaryMessage
  | CompactionSummaryMessage;
```

## エントリの基底

すべてのエントリ（`SessionHeader` を除く）は `SessionEntryBase` を継承します。

```typescript
interface SessionEntryBase {
  type: string;
  id: string;           // 8-char hex ID
  parentId: string | null;  // 親エントリ ID（最初のエントリでは null）
  timestamp: string;    // ISO timestamp
}
```

## エントリ型

### SessionHeader

ファイルの 1 行目です。メタデータのみで、ツリーの一部ではありません（`id` / `parentId` なし）。

```json
{"type":"session","version":3,"id":"uuid","timestamp":"2024-12-03T14:00:00.000Z","cwd":"/path/to/project"}
```

親を持つセッション（`/fork`、`/clone`、または `newSession({ parentSession })` で作成）では次のようになります。

```json
{"type":"session","version":3,"id":"uuid","timestamp":"2024-12-03T14:00:00.000Z","cwd":"/path/to/project","parentSession":"/path/to/original/session.jsonl"}
```

### SessionMessageEntry

会話中の 1 メッセージです。`message` フィールドに `AgentMessage` が入ります。

```json
{"type":"message","id":"a1b2c3d4","parentId":"prev1234","timestamp":"2024-12-03T14:00:01.000Z","message":{"role":"user","content":"Hello"}}
{"type":"message","id":"b2c3d4e5","parentId":"a1b2c3d4","timestamp":"2024-12-03T14:00:02.000Z","message":{"role":"assistant","content":[{"type":"text","text":"Hi!"}],"provider":"anthropic","model":"claude-sonnet-4-5","usage":{...},"stopReason":"stop"}}
{"type":"message","id":"c3d4e5f6","parentId":"b2c3d4e5","timestamp":"2024-12-03T14:00:03.000Z","message":{"role":"toolResult","toolCallId":"call_123","toolName":"bash","content":[{"type":"text","text":"output"}],"isError":false}}
```

### ModelChangeEntry

ユーザーがセッション途中でモデルを切り替えたときに出力されます。

```json
{"type":"model_change","id":"d4e5f6g7","parentId":"c3d4e5f6","timestamp":"2024-12-03T14:05:00.000Z","provider":"openai","modelId":"gpt-4o"}
```

### ThinkingLevelChangeEntry

ユーザーが thinking / reasoning レベルを変更したときに出力されます。

```json
{"type":"thinking_level_change","id":"e5f6g7h8","parentId":"d4e5f6g7","timestamp":"2024-12-03T14:06:00.000Z","thinkingLevel":"high"}
```

### CompactionEntry

コンテキストがコンパクト化されたときに作成されます。以前のメッセージの要約を保存します。

```json
{"type":"compaction","id":"f6g7h8i9","parentId":"e5f6g7h8","timestamp":"2024-12-03T14:10:00.000Z","summary":"User discussed X, Y, Z...","firstKeptEntryId":"c3d4e5f6","tokensBefore":50000}
```

任意フィールド:
- `details`: 実装固有データ（デフォルトでは `{ readFiles: string[], modifiedFiles: string[] }`、拡張機能では独自データ）
- `fromHook`: 拡張機能生成なら `true`、pi 生成なら `false` / `undefined`（レガシーなフィールド名）

### BranchSummaryEntry

`/tree` でブランチを切り替える際、共通祖先までの離脱ブランチを LLM が要約したときに作成されます。離れる経路の文脈を保持します。

```json
{"type":"branch_summary","id":"g7h8i9j0","parentId":"a1b2c3d4","timestamp":"2024-12-03T14:15:00.000Z","fromId":"f6g7h8i9","summary":"Branch explored approach A..."}
```

任意フィールド:
- `details`: デフォルトではファイル追跡データ（`{ readFiles: string[], modifiedFiles: string[] }`）、拡張機能では独自データ
- `fromHook`: 拡張機能生成なら `true`、pi 生成なら `false` / `undefined`（レガシーなフィールド名）

### CustomEntry

拡張機能の状態永続化です。LLM コンテキストには参加しません。

```json
{"type":"custom","id":"h8i9j0k1","parentId":"g7h8i9j0","timestamp":"2024-12-03T14:20:00.000Z","customType":"my-extension","data":{"count":42}}
```

再読み込み時に自分の拡張機能エントリを識別するには `customType` を使ってください。

### CustomMessageEntry

LLM コンテキストには参加する、拡張機能注入メッセージです。

```json
{"type":"custom_message","id":"i9j0k1l2","parentId":"h8i9j0k1","timestamp":"2024-12-03T14:25:00.000Z","customType":"my-extension","content":"Injected context...","display":true}
```

フィールド:
- `content`: 文字列または `(TextContent | ImageContent)[]`（`UserMessage` と同じ）
- `display`: `true` なら独自スタイルで TUI に表示、`false` なら非表示
- `details`: 任意の拡張機能固有メタデータ（LLM には送られない）

### LabelEntry

エントリに対するユーザー定義のブックマーク / マーカーです。

```json
{"type":"label","id":"j0k1l2m3","parentId":"i9j0k1l2","timestamp":"2024-12-03T14:30:00.000Z","targetId":"a1b2c3d4","label":"checkpoint-1"}
```

ラベルを消すには `label` を `undefined` に設定します。

### SessionInfoEntry

セッションメタデータ（例: ユーザー定義の表示名）です。`/name`、`--name` / `-n`、または拡張機能内の `pi.setSessionName()` で設定します。

```json
{"type":"session_info","id":"k1l2m3n4","parentId":"j0k1l2m3","timestamp":"2024-12-03T14:35:00.000Z","name":"Refactor auth module"}
```

セッション名が設定されている場合、セッションセレクタ（`/resume`）では最初のメッセージの代わりにそれが表示されます。

## ツリー構造

エントリはツリーを形成します。
- 最初のエントリは `parentId: null`
- 以降の各エントリは `parentId` を通じて親を指す
- 分岐では以前のエントリから新しい子が作られる
- 「leaf」がツリー内の現在位置

```
[user msg] ─── [assistant] ─── [user msg] ─── [assistant] ─┬─ [user msg] ← current leaf
                                                            │
                                                            └─ [branch_summary] ─── [user msg] ← alternate branch
```

## コンテキスト構築

`buildSessionContext()` は現在の leaf から root までたどり、LLM 用のメッセージ一覧を生成します。

1. 経路上の全エントリを収集する
2. 現在のモデル設定と thinking level 設定を抽出する
3. 経路上に `CompactionEntry` がある場合:
   - まず要約を出力する
   - 次に `firstKeptEntryId` から compaction までのメッセージを出力する
   - その後、compaction 以降のメッセージを出力する
4. `BranchSummaryEntry` と `CustomMessageEntry` を適切なメッセージ形式へ変換する

## パース例

```typescript
import { readFileSync } from "fs";

const lines = readFileSync("session.jsonl", "utf8").trim().split("\n");

for (const line of lines) {
  const entry = JSON.parse(line);

  switch (entry.type) {
    case "session":
      console.log(`Session v${entry.version ?? 1}: ${entry.id}`);
      break;
    case "message":
      console.log(`[${entry.id}] ${entry.message.role}: ${JSON.stringify(entry.message.content)}`);
      break;
    case "compaction":
      console.log(`[${entry.id}] Compaction: ${entry.tokensBefore} tokens summarized`);
      break;
    case "branch_summary":
      console.log(`[${entry.id}] Branch from ${entry.fromId}`);
      break;
    case "custom":
      console.log(`[${entry.id}] Custom (${entry.customType}): ${JSON.stringify(entry.data)}`);
      break;
    case "custom_message":
      console.log(`[${entry.id}] Extension message (${entry.customType}): ${entry.content}`);
      break;
    case "label":
      console.log(`[${entry.id}] Label "${entry.label}" on ${entry.targetId}`);
      break;
    case "model_change":
      console.log(`[${entry.id}] Model: ${entry.provider}/${entry.modelId}`);
      break;
    case "thinking_level_change":
      console.log(`[${entry.id}] Thinking: ${entry.thinkingLevel}`);
      break;
  }
}
```

## SessionManager API

プログラムからセッションを扱うための主要メソッドです。

### 静的作成メソッド
- `SessionManager.create(cwd, sessionDir?)` - 新しいセッション
- `SessionManager.open(path, sessionDir?)` - 既存セッションファイルを開く
- `SessionManager.continueRecent(cwd, sessionDir?)` - もっとも新しいセッションを続行、なければ新規作成
- `SessionManager.inMemory(cwd?)` - ファイル永続化なし
- `SessionManager.forkFrom(sourcePath, targetCwd, sessionDir?)` - 別プロジェクトのセッションから fork

### 静的列挙メソッド
- `SessionManager.list(cwd, sessionDir?, onProgress?)` - あるディレクトリのセッション一覧
- `SessionManager.listAll(onProgress?)` - すべてのプロジェクトのセッション一覧

### インスタンスメソッド - セッション管理
- `newSession(options?)` - 新しいセッションを開始（options: `{ parentSession?: string }`）
- `setSessionFile(path)` - 別のセッションファイルへ切り替える
- `createBranchedSession(leafId)` - ブランチを新しいセッションファイルへ抽出する

### インスタンスメソッド - 追記系（すべてエントリ ID を返す）
- `appendMessage(message)` - メッセージを追加
- `appendThinkingLevelChange(level)` - thinking 変更を記録
- `appendModelChange(provider, modelId)` - モデル変更を記録
- `appendCompaction(summary, firstKeptEntryId, tokensBefore, details?, fromHook?)` - compaction を追加
- `appendCustomEntry(customType, data?)` - 拡張機能状態（コンテキスト外）
- `appendSessionInfo(name)` - セッション表示名を設定
- `appendCustomMessageEntry(customType, content, display, details?)` - 拡張機能メッセージ（コンテキスト内）
- `appendLabelChange(targetId, label)` - ラベル設定 / 解除

### インスタンスメソッド - ツリーナビゲーション
- `getLeafId()` - 現在位置を取得
- `getLeafEntry()` - 現在の leaf エントリを取得
- `getEntry(id)` - ID でエントリ取得
- `getBranch(fromId?)` - 指定エントリから root までたどる
- `getTree()` - ツリー全体を取得
- `getChildren(parentId)` - 直接の子を取得
- `getLabel(id)` - エントリのラベル取得
- `branch(entryId)` - leaf を以前のエントリへ移動
- `resetLeaf()` - leaf を null に戻す（どのエントリより前）
- `branchWithSummary(entryId, summary, details?, fromHook?)` - 文脈要約付きで branch

### インスタンスメソッド - コンテキストと情報
- `buildSessionContext()` - LLM 用の messages、thinkingLevel、model を取得
- `getEntries()` - 全エントリ（ヘッダー除く）
- `getHeader()` - セッションヘッダーメタデータを取得
- `getSessionName()` - 最新の session_info エントリから表示名を取得
- `getCwd()` - 作業ディレクトリ
- `getSessionDir()` - セッション保存ディレクトリ
- `getSessionId()` - セッション UUID
- `getSessionFile()` - セッションファイルパス（インメモリなら undefined）
- `isPersisted()` - ディスクへ保存済みかどうか
