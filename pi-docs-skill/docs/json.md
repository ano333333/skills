# JSON イベントストリームモード

> 要約: JSON Event Stream Mode は、Pi のセッションイベントを標準出力へ JSON Lines 形式で流すためのモードです。外部ツールや独自 UI から Pi を組み込む際に、イベント種別・メッセージ型・出力フォーマットを把握するための基礎資料になります。最初のセッションヘッダーから各種イベントの流れ、実用的なコマンド例までを簡潔に整理しています。

```bash
pi --mode json "Your prompt"
```

セッション中のすべてのイベントを JSON Lines として stdout に出力します。Pi を他のツールやカスタム UI に統合する際に便利です。

## Event Types

イベントは [`AgentSessionEvent`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/agent-session.ts#L102) で定義されています。

```typescript
type AgentSessionEvent =
  | AgentEvent
  | { type: "queue_update"; steering: readonly string[]; followUp: readonly string[] }
  | { type: "compaction_start"; reason: "manual" | "threshold" | "overflow" }
  | { type: "compaction_end"; reason: "manual" | "threshold" | "overflow"; result: CompactionResult | undefined; aborted: boolean; willRetry: boolean; errorMessage?: string }
  | { type: "auto_retry_start"; attempt: number; maxAttempts: number; delayMs: number; errorMessage: string }
  | { type: "auto_retry_end"; success: boolean; attempt: number; finalError?: string };
```

`queue_update` は、保留中の steering キューと follow-up キューが変更されるたびに、その全体を出力します。`compaction_start` と `compaction_end` は、手動・自動の両方の compaction を対象にします。

[`AgentEvent`](https://github.com/earendil-works/pi-mono/blob/main/packages/agent/src/types.ts#L179) 由来の基本イベント:

```typescript
type AgentEvent =
  // Agent lifecycle
  | { type: "agent_start" }
  | { type: "agent_end"; messages: AgentMessage[] }
  // Turn lifecycle
  | { type: "turn_start" }
  | { type: "turn_end"; message: AgentMessage; toolResults: ToolResultMessage[] }
  // Message lifecycle
  | { type: "message_start"; message: AgentMessage }
  | { type: "message_update"; message: AgentMessage; assistantMessageEvent: AssistantMessageEvent }
  | { type: "message_end"; message: AgentMessage }
  // Tool execution
  | { type: "tool_execution_start"; toolCallId: string; toolName: string; args: any }
  | { type: "tool_execution_update"; toolCallId: string; toolName: string; args: any; partialResult: any }
  | { type: "tool_execution_end"; toolCallId: string; toolName: string; result: any; isError: boolean };
```

## Message Types

[`packages/ai/src/types.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/ai/src/types.ts#L134) にある基本メッセージ:
- `UserMessage`（134 行目）
- `AssistantMessage`（140 行目）
- `ToolResultMessage`（152 行目）

[`packages/coding-agent/src/core/messages.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/messages.ts#L29) にある拡張メッセージ:
- `BashExecutionMessage`（29 行目）
- `CustomMessage`（46 行目）
- `BranchSummaryMessage`（55 行目）
- `CompactionSummaryMessage`（62 行目）

## Output Format

各行は JSON オブジェクトです。最初の 1 行はセッションヘッダーです。

```json
{"type":"session","version":3,"id":"uuid","timestamp":"...","cwd":"/path"}
```

その後、イベントが発生するたびに続けて出力されます。

```json
{"type":"agent_start"}
{"type":"turn_start"}
{"type":"message_start","message":{"role":"assistant","content":[],...}}
{"type":"message_update","message":{...},"assistantMessageEvent":{"type":"text_delta","delta":"Hello",...}}
{"type":"message_end","message":{...}}
{"type":"turn_end","message":{...},"toolResults":[]}
{"type":"agent_end","messages":[...]}
```

## Example

```bash
pi --mode json "List files" 2>/dev/null | jq -c 'select(.type == "message_end")'
```
