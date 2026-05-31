# RPC モード

> 要約: RPC Mode は、stdin/stdout 上の JSON プロトコル経由で Pi コーディングエージェントをヘッドレス運用するためのモードです。コマンド、イベント、拡張 UI サブプロトコル、主要データ型までを網羅しており、IDE や独自クライアントへ組み込む際の実装基準になります。特に JSONL framing、キュー制御、セッション管理、イベント相関の扱いが重要です。

RPC mode は、stdin/stdout 上の JSON プロトコルを介してコーディングエージェントをヘッドレスで動作させるためのモードです。これは、他のアプリケーション、IDE、または独自 UI にエージェントを組み込む際に有用です。

**Node.js/TypeScript 利用者向けメモ**: Node.js アプリケーションを構築している場合は、サブプロセスを起動する代わりに `@earendil-works/pi-coding-agent` から `AgentSession` を直接使うことを検討してください。API は [`src/core/agent-session.ts`](../src/core/agent-session.ts) を参照してください。サブプロセス方式の TypeScript クライアント例は [`src/modes/rpc/rpc-client.ts`](../src/modes/rpc/rpc-client.ts) を参照してください。

## Starting RPC Mode

```bash
pi --mode rpc [options]
```

よく使うオプション:
- `--provider <name>`: LLM プロバイダを設定します（anthropic、openai、google など）
- `--model <pattern>`: モデルのパターンまたは ID を設定します（`provider/id` と任意の `:<thinking>` をサポート）
- `--name <name>` / `-n <name>`: 起動時にセッション表示名を設定します
- `--no-session`: セッション永続化を無効にします
- `--session-dir <path>`: セッション保存ディレクトリを指定します

## Protocol Overview

- **Commands**: stdin に 1 行ずつ送る JSON オブジェクト
- **Responses**: コマンドの成功/失敗を示す `type: "response"` の JSON オブジェクト
- **Events**: stdout に JSON Lines としてストリーミングされるエージェントイベント

すべてのコマンドは、リクエストとレスポンスの対応付け用に任意の `id` フィールドをサポートします。指定した場合、対応するレスポンスにも同じ `id` が含まれます。

### Framing

RPC mode は、レコード区切り文字を LF（`\n`）のみに限定した厳密な JSONL セマンティクスを使用します。

これはクライアント実装上、次の点で重要です。
- レコード分割は `\n` のみで行う
- 末尾の `\r` を取り除くことで、任意の `\r\n` 入力も受け付ける
- Unicode の区切り文字を改行として扱う汎用行リーダーは使わない

特に Node の `readline` は、JSON 文字列内で有効な `U+2028` と `U+2029` でも分割してしまうため、RPC mode ではプロトコル準拠ではありません。

## Commands

### Prompting

#### prompt

ユーザープロンプトをエージェントへ送信します。コマンドレスポンスは、プロンプトが受理・キュー投入・処理された後に出力されます。受理後もイベントのストリーミングは非同期で継続します。

```json
{"id": "req-1", "type": "prompt", "message": "Hello, world!"}
```

画像付き:
```json
{"type": "prompt", "message": "What's in this image?", "images": [{"type": "image", "data": "base64-encoded-data", "mimeType": "image/png"}]}
```

**ストリーミング中**: すでにエージェントがストリーミング中の場合、メッセージをキューに積むには `streamingBehavior` の指定が必要です。

```json
{"type": "prompt", "message": "New instruction", "streamingBehavior": "steer"}
```

- `"steer"`: エージェント実行中にメッセージをキュー投入します。現在の assistant turn がツール呼び出しを完了した後、次の LLM 呼び出しの前に配信されます。
- `"followUp"`: エージェントが終了するまで待機します。メッセージは、エージェント停止時にのみ配信されます。

エージェントがストリーミング中で、`streamingBehavior` が指定されていない場合、このコマンドはエラーになります。

**Extension command**: メッセージが拡張コマンド（例: `/mycommand`）である場合、ストリーミング中でも即座に実行されます。拡張コマンドは `pi.sendMessage()` を通じて独自に LLM 連携を管理します。

**入力展開**: Skill command（`/skill:name`）と prompt template（`/template`）は、送信またはキュー投入の前に展開されます。

レスポンス:
```json
{"id": "req-1", "type": "response", "command": "prompt", "success": true}
```

`success: true` は、プロンプトが受理された、キューに入った、または即時処理されたことを意味します。`success: false` は、受理前に拒否されたことを意味します。受理後に発生した失敗は、同じリクエスト id に対する 2 つ目の `response` ではなく、通常のイベントやメッセージストリームを通じて報告されます。

`images` フィールドは任意です。各画像は `ImageContent` 形式を使用します: `{"type": "image", "data": "base64-encoded-data", "mimeType": "image/png"}`。

#### steer

エージェント実行中に steering メッセージをキュー投入します。現在の assistant turn がツール呼び出しを完了した後、次の LLM 呼び出しの前に配信されます。Skill command と prompt template は展開されます。Extension command は使えません（代わりに `prompt` を使ってください）。

```json
{"type": "steer", "message": "Stop and do this instead"}
```

画像付き:
```json
{"type": "steer", "message": "Look at this instead", "images": [{"type": "image", "data": "base64-encoded-data", "mimeType": "image/png"}]}
```

`images` フィールドは任意です。各画像は `ImageContent` 形式を使用します（`prompt` と同じです）。

レスポンス:
```json
{"type": "response", "command": "steer", "success": true}
```

steering メッセージの処理方法は [set_steering_mode](#set_steering_mode) を参照してください。

#### follow_up

エージェント完了後に処理される follow-up メッセージをキュー投入します。エージェントに残りのツール呼び出しや steering メッセージがなくなった時点でのみ配信されます。Skill command と prompt template は展開されます。Extension command は使えません（代わりに `prompt` を使ってください）。

```json
{"type": "follow_up", "message": "After you're done, also do this"}
```

画像付き:
```json
{"type": "follow_up", "message": "Also check this image", "images": [{"type": "image", "data": "base64-encoded-data", "mimeType": "image/png"}]}
```

`images` フィールドは任意です。各画像は `ImageContent` 形式を使用します（`prompt` と同じです）。

レスポンス:
```json
{"type": "response", "command": "follow_up", "success": true}
```

follow-up メッセージの処理方法は [set_follow_up_mode](#set_follow_up_mode) を参照してください。

#### abort

現在のエージェント処理を中断します。

```json
{"type": "abort"}
```

レスポンス:
```json
{"type": "response", "command": "abort", "success": true}
```

#### new_session

新しいセッションを開始します。`session_before_switch` 拡張イベントハンドラによってキャンセルされる場合があります。

```json
{"type": "new_session"}
```

親セッション追跡を任意で付ける場合:
```json
{"type": "new_session", "parentSession": "/path/to/parent-session.jsonl"}
```

レスポンス:
```json
{"type": "response", "command": "new_session", "success": true, "data": {"cancelled": false}}
```

拡張によってキャンセルされた場合:
```json
{"type": "response", "command": "new_session", "success": true, "data": {"cancelled": true}}
```

### State

#### get_state

現在のセッション状態を取得します。

```json
{"type": "get_state"}
```

レスポンス:
```json
{
  "type": "response",
  "command": "get_state",
  "success": true,
  "data": {
    "model": {...},
    "thinkingLevel": "medium",
    "isStreaming": false,
    "isCompacting": false,
    "steeringMode": "all",
    "followUpMode": "one-at-a-time",
    "sessionFile": "/path/to/session.jsonl",
    "sessionId": "abc123",
    "sessionName": "my-feature-work",
    "autoCompactionEnabled": true,
    "messageCount": 5,
    "pendingMessageCount": 0
  }
}
```

`model` フィールドは完全な [Model](#model) オブジェクト、または `null` です。`sessionName` フィールドは `set_session_name` で設定した表示名で、未設定の場合は省略されます。

#### get_messages

会話内のすべてのメッセージを取得します。

```json
{"type": "get_messages"}
```

レスポンス:
```json
{
  "type": "response",
  "command": "get_messages",
  "success": true,
  "data": {"messages": [...]}
}
```

メッセージは `AgentMessage` オブジェクトです（[Message Types](#message-types) を参照）。

### Model

#### set_model

特定のモデルへ切り替えます。

```json
{"type": "set_model", "provider": "anthropic", "modelId": "claude-sonnet-4-20250514"}
```

レスポンスには完全な [Model](#model) オブジェクトが含まれます。
```json
{
  "type": "response",
  "command": "set_model",
  "success": true,
  "data": {...}
}
```

#### cycle_model

次に利用可能なモデルへ切り替えます。利用可能なモデルが 1 つだけの場合、`data` は `null` を返します。

```json
{"type": "cycle_model"}
```

レスポンス:
```json
{
  "type": "response",
  "command": "cycle_model",
  "success": true,
  "data": {
    "model": {...},
    "thinkingLevel": "medium",
    "isScoped": false
  }
}
```

`model` フィールドは完全な [Model](#model) オブジェクトです。

#### get_available_models

設定済みの全モデルを一覧表示します。

```json
{"type": "get_available_models"}
```

レスポンスには完全な [Model](#model) オブジェクトの配列が含まれます。
```json
{
  "type": "response",
  "command": "get_available_models",
  "success": true,
  "data": {
    "models": [...]
  }
}
```

### Thinking

#### set_thinking_level

thinking をサポートするモデルに対して、推論レベルを設定します。

```json
{"type": "set_thinking_level", "level": "high"}
```

レベル: `"off"`, `"minimal"`, `"low"`, `"medium"`, `"high"`, `"xhigh"`

注: `"xhigh"` は OpenAI の codex-max モデルでのみサポートされます。

レスポンス:
```json
{"type": "response", "command": "set_thinking_level", "success": true}
```

#### cycle_thinking_level

利用可能な thinking レベルを順に切り替えます。モデルが thinking をサポートしていない場合、`data` は `null` を返します。

```json
{"type": "cycle_thinking_level"}
```

レスポンス:
```json
{
  "type": "response",
  "command": "cycle_thinking_level",
  "success": true,
  "data": {"level": "high"}
}
```

### Queue Modes

#### set_steering_mode

steering メッセージ（`steer` 由来）の配信方法を制御します。

```json
{"type": "set_steering_mode", "mode": "one-at-a-time"}
```

モード:
- `"all"`: 現在の assistant turn がツール呼び出しを終えたあと、すべての steering メッセージを配信します
- `"one-at-a-time"`: 完了した assistant turn ごとに 1 件の steering メッセージを配信します（デフォルト）

レスポンス:
```json
{"type": "response", "command": "set_steering_mode", "success": true}
```

#### set_follow_up_mode

follow-up メッセージ（`follow_up` 由来）の配信方法を制御します。

```json
{"type": "set_follow_up_mode", "mode": "one-at-a-time"}
```

モード:
- `"all"`: エージェント完了時に、すべての follow-up メッセージを配信します
- `"one-at-a-time"`: エージェント完了ごとに 1 件の follow-up メッセージを配信します（デフォルト）

レスポンス:
```json
{"type": "response", "command": "set_follow_up_mode", "success": true}
```

### Compaction

#### compact

トークン使用量を減らすため、会話コンテキストを手動で compaction します。

```json
{"type": "compact"}
```

カスタム命令付き:
```json
{"type": "compact", "customInstructions": "Focus on code changes"}
```

レスポンス:
```json
{
  "type": "response",
  "command": "compact",
  "success": true,
  "data": {
    "summary": "Summary of conversation...",
    "firstKeptEntryId": "abc123",
    "tokensBefore": 150000,
    "details": {}
  }
}
```

#### set_auto_compaction

コンテキストがほぼ一杯になった際の自動 compaction を有効または無効にします。

```json
{"type": "set_auto_compaction", "enabled": true}
```

レスポンス:
```json
{"type": "response", "command": "set_auto_compaction", "success": true}
```

### Retry

#### set_auto_retry

一時的なエラー（overloaded、rate limit、5xx）発生時の自動再試行を有効または無効にします。

```json
{"type": "set_auto_retry", "enabled": true}
```

レスポンス:
```json
{"type": "response", "command": "set_auto_retry", "success": true}
```

#### abort_retry

進行中の再試行を中断します（待機を取り消し、再試行を停止します）。

```json
{"type": "abort_retry"}
```

レスポンス:
```json
{"type": "response", "command": "abort_retry", "success": true}
```

### Bash

#### bash

シェルコマンドを実行し、その出力を会話コンテキストに追加します。

```json
{"type": "bash", "command": "ls -la"}
```

レスポンス:
```json
{
  "type": "response",
  "command": "bash",
  "success": true,
  "data": {
    "output": "total 48\ndrwxr-xr-x ...",
    "exitCode": 0,
    "cancelled": false,
    "truncated": false
  }
}
```

出力が切り詰められた場合、`fullOutputPath` が含まれます。
```json
{
  "type": "response",
  "command": "bash",
  "success": true,
  "data": {
    "output": "truncated output...",
    "exitCode": 0,
    "cancelled": false,
    "truncated": true,
    "fullOutputPath": "/tmp/pi-bash-abc123.log"
  }
}
```

**bash の結果が LLM に届くまで**

`bash` コマンドは即座に実行され、`BashResult` を返します。内部的には `BashExecutionMessage` が作成され、エージェントのメッセージ状態に保存されます。このメッセージ自体はイベントを発行しません。

次の `prompt` コマンドが送信されると、すべてのメッセージ（`BashExecutionMessage` を含む）が LLM へ送る前に変換されます。`BashExecutionMessage` は、次の形式の `UserMessage` に変換されます。

````
Ran `ls -la`
```
total 48
drwxr-xr-x ...
```
````

これは次を意味します。
1. bash の出力は即時ではなく、**次の prompt** で LLM コンテキストに含まれます
2. prompt の前に複数の bash コマンドを実行でき、その出力はすべて含まれます
3. `BashExecutionMessage` 自体に対するイベントは発行されません

#### abort_bash

実行中の bash コマンドを中断します。

```json
{"type": "abort_bash"}
```

レスポンス:
```json
{"type": "response", "command": "abort_bash", "success": true}
```

### Session

#### get_session_stats

トークン使用量、コスト統計、および現在のコンテキストウィンドウ使用量を取得します。

```json
{"type": "get_session_stats"}
```

レスポンス:
```json
{
  "type": "response",
  "command": "get_session_stats",
  "success": true,
  "data": {
    "sessionFile": "/path/to/session.jsonl",
    "sessionId": "abc123",
    "userMessages": 5,
    "assistantMessages": 5,
    "toolCalls": 12,
    "toolResults": 12,
    "totalMessages": 22,
    "tokens": {
      "input": 50000,
      "output": 10000,
      "cacheRead": 40000,
      "cacheWrite": 5000,
      "total": 105000
    },
    "cost": 0.45,
    "contextUsage": {
      "tokens": 60000,
      "contextWindow": 200000,
      "percent": 30
    }
  }
}
```

`tokens` には、現在のセッション状態に対する assistant 使用量の合計が入ります。`contextUsage` には、compaction 判定やフッター表示に使われる実際の現在コンテキストウィンドウ推定値が入ります。

`contextUsage` は、モデルまたはコンテキストウィンドウが利用できない場合は省略されます。`contextUsage.tokens` と `contextUsage.percent` は、compaction 直後は `null` になります。これは、compaction 後の新しい assistant 応答が有効な使用量データを返すまで確定できないためです。

#### export_html

セッションを HTML ファイルとしてエクスポートします。

```json
{"type": "export_html"}
```

カスタムパス指定:
```json
{"type": "export_html", "outputPath": "/tmp/session.html"}
```

レスポンス:
```json
{
  "type": "response",
  "command": "export_html",
  "success": true,
  "data": {"path": "/tmp/session.html"}
}
```

#### switch_session

別のセッションファイルを読み込みます。`session_before_switch` 拡張イベントハンドラによってキャンセルされる場合があります。

```json
{"type": "switch_session", "sessionPath": "/path/to/session.jsonl"}
```

レスポンス:
```json
{"type": "response", "command": "switch_session", "success": true, "data": {"cancelled": false}}
```

拡張によって切り替えがキャンセルされた場合:
```json
{"type": "response", "command": "switch_session", "success": true, "data": {"cancelled": true}}
```

#### fork

アクティブブランチ上の過去のユーザーメッセージから新しい fork を作成します。`session_before_fork` 拡張イベントハンドラによってキャンセルされる場合があります。戻り値には fork 元メッセージのテキストが含まれます。

```json
{"type": "fork", "entryId": "abc123"}
```

レスポンス:
```json
{
  "type": "response",
  "command": "fork",
  "success": true,
  "data": {"text": "The original prompt text...", "cancelled": false}
}
```

拡張によって fork がキャンセルされた場合:
```json
{
  "type": "response",
  "command": "fork",
  "success": true,
  "data": {"text": "The original prompt text...", "cancelled": true}
}
```

#### clone

現在の位置にあるアクティブブランチを、新しいセッションとして複製します。`session_before_fork` 拡張イベントハンドラによってキャンセルされる場合があります。

```json
{"type": "clone"}
```

レスポンス:
```json
{
  "type": "response",
  "command": "clone",
  "success": true,
  "data": {"cancelled": false}
}
```

拡張によって clone がキャンセルされた場合:
```json
{
  "type": "response",
  "command": "clone",
  "success": true,
  "data": {"cancelled": true}
}
```

#### get_fork_messages

fork 対象として利用可能なユーザーメッセージを取得します。

```json
{"type": "get_fork_messages"}
```

レスポンス:
```json
{
  "type": "response",
  "command": "get_fork_messages",
  "success": true,
  "data": {
    "messages": [
      {"entryId": "abc123", "text": "First prompt..."},
      {"entryId": "def456", "text": "Second prompt..."}
    ]
  }
}
```

#### get_last_assistant_text

直近の assistant メッセージのテキスト内容を取得します。

```json
{"type": "get_last_assistant_text"}
```

レスポンス:
```json
{
  "type": "response",
  "command": "get_last_assistant_text",
  "success": true,
  "data": {"text": "The assistant's response..."}
}
```

assistant メッセージが存在しない場合は `{"text": null}` を返します。

#### set_session_name

現在のセッションに表示名を設定します。この名前はセッション一覧に表示され、識別に役立ちます。

```json
{"type": "set_session_name", "name": "my-feature-work"}
```

レスポンス:
```json
{
  "type": "response",
  "command": "set_session_name",
  "success": true
}
```

現在のセッション名は `get_state` の `sessionName` フィールドで取得できます。RPC mode 起動時に初期名を設定するには、`pi --mode rpc` 実行時に `--name <name>` または `-n <name>` を指定してください。

### Commands

#### get_commands

利用可能なコマンド（extension command、prompt template、skill）を取得します。これらは `prompt` コマンドで `/` を付けて呼び出せます。

```json
{"type": "get_commands"}
```

レスポンス:
```json
{
  "type": "response",
  "command": "get_commands",
  "success": true,
  "data": {
    "commands": [
      {"name": "session-name", "description": "Set or clear session name", "source": "extension", "path": "/home/user/.pi/agent/extensions/session.ts"},
      {"name": "fix-tests", "description": "Fix failing tests", "source": "prompt", "location": "project", "path": "/home/user/myproject/.pi/agent/prompts/fix-tests.md"},
      {"name": "skill:brave-search", "description": "Web search via Brave API", "source": "skill", "location": "user", "path": "/home/user/.pi/agent/skills/brave-search/SKILL.md"}
    ]
  }
}
```

各コマンドは次の情報を持ちます。
- `name`: コマンド名（`/name` で呼び出す）
- `description`: 人間向けの説明（extension command では省略される場合があります）
- `source`: コマンドの種別
  - `"extension"`: 拡張内の `pi.registerCommand()` で登録されたもの
  - `"prompt"`: prompt template の `.md` ファイルから読み込まれたもの
  - `"skill"`: skill ディレクトリから読み込まれたもの（名前には `skill:` が付きます）
- `location`: どこから読み込まれたか（任意。拡張には含まれません）
  - `"user"`: ユーザーレベル（`~/.pi/agent/`）
  - `"project"`: プロジェクトレベル（`./.pi/agent/`）
  - `"path"`: CLI や設定で明示指定されたパス
- `path`: コマンドソースの絶対ファイルパス（任意）

**注**: 組み込み TUI コマンド（`/settings`、`/hotkeys` など）は含まれません。これらは対話モード専用で処理されるため、`prompt` 経由で送っても実行されません。

## Events

イベントは、エージェント動作中に stdout へ JSON Lines としてストリーミングされます。イベントには `id` フィールドは含まれません（`id` があるのはレスポンスのみです）。

### Event Types

| Event | Description |
|-------|-------------|
| `agent_start` | エージェントが処理を開始 |
| `agent_end` | エージェントが完了（生成したすべてのメッセージを含む） |
| `turn_start` | 新しい turn が開始 |
| `turn_end` | turn が完了（assistant メッセージと tool result を含む） |
| `message_start` | メッセージ生成を開始 |
| `message_update` | ストリーミング更新（text/thinking/toolcall の差分） |
| `message_end` | メッセージ生成を完了 |
| `tool_execution_start` | ツール実行を開始 |
| `tool_execution_update` | ツール実行の進捗（ストリーミング出力） |
| `tool_execution_end` | ツール実行を完了 |
| `queue_update` | 保留中の steering/follow-up キューが変更 |
| `compaction_start` | compaction 開始 |
| `compaction_end` | compaction 完了 |
| `auto_retry_start` | 自動再試行を開始（一時エラー後） |
| `auto_retry_end` | 自動再試行を完了（成功または最終失敗） |
| `extension_error` | 拡張がエラーを投げた |

### agent_start

プロンプト処理をエージェントが開始したときに発行されます。

```json
{"type": "agent_start"}
```

### agent_end

エージェントが完了したときに発行されます。この実行中に生成されたすべてのメッセージを含みます。

```json
{
  "type": "agent_end",
  "messages": [...]
}
```

### turn_start / turn_end

1 つの turn は、1 回の assistant 応答と、それに伴って発生するツール呼び出しおよび結果から構成されます。

```json
{"type": "turn_start"}
```

```json
{
  "type": "turn_end",
  "message": {...},
  "toolResults": [...]
}
```

### message_start / message_end

メッセージの開始時と完了時に発行されます。`message` フィールドには `AgentMessage` が入ります。

```json
{"type": "message_start", "message": {...}}
{"type": "message_end", "message": {...}}
```

### message_update (Streaming)

assistant メッセージのストリーミング中に発行されます。部分的なメッセージと、ストリーミング差分イベントの両方を含みます。

```json
{
  "type": "message_update",
  "message": {...},
  "assistantMessageEvent": {
    "type": "text_delta",
    "contentIndex": 0,
    "delta": "Hello ",
    "partial": {...}
  }
}
```

`assistantMessageEvent` フィールドには、次のいずれかの差分タイプが入ります。

| Type | Description |
|------|-------------|
| `start` | メッセージ生成開始 |
| `text_start` | テキスト content block 開始 |
| `text_delta` | テキスト content chunk |
| `text_end` | テキスト content block 終了 |
| `thinking_start` | thinking block 開始 |
| `thinking_delta` | thinking content chunk |
| `thinking_end` | thinking block 終了 |
| `toolcall_start` | tool call 開始 |
| `toolcall_delta` | tool call 引数 chunk |
| `toolcall_end` | tool call 終了（完全な `toolCall` オブジェクトを含む） |
| `done` | メッセージ完了（reason: `"stop"`, `"length"`, `"toolUse"`） |
| `error` | エラー発生（reason: `"aborted"`, `"error"`） |

テキスト応答のストリーミング例:
```json
{"type":"message_update","message":{...},"assistantMessageEvent":{"type":"text_start","contentIndex":0,"partial":{...}}}
{"type":"message_update","message":{...},"assistantMessageEvent":{"type":"text_delta","contentIndex":0,"delta":"Hello","partial":{...}}}
{"type":"message_update","message":{...},"assistantMessageEvent":{"type":"text_delta","contentIndex":0,"delta":" world","partial":{...}}}
{"type":"message_update","message":{...},"assistantMessageEvent":{"type":"text_end","contentIndex":0,"content":"Hello world","partial":{...}}}
```

### tool_execution_start / tool_execution_update / tool_execution_end

ツールの開始時、進捗ストリーミング中、完了時に発行されます。

```json
{
  "type": "tool_execution_start",
  "toolCallId": "call_abc123",
  "toolName": "bash",
  "args": {"command": "ls -la"}
}
```

実行中は、`tool_execution_update` イベントが部分結果をストリーミングします（例: bash の出力が到着するたび）。

```json
{
  "type": "tool_execution_update",
  "toolCallId": "call_abc123",
  "toolName": "bash",
  "args": {"command": "ls -la"},
  "partialResult": {
    "content": [{"type": "text", "text": "partial output so far..."}],
    "details": {"truncation": null, "fullOutputPath": null}
  }
}
```

完了時:

```json
{
  "type": "tool_execution_end",
  "toolCallId": "call_abc123",
  "toolName": "bash",
  "result": {
    "content": [{"type": "text", "text": "total 48\n..."}],
    "details": {...}
  },
  "isError": false
}
```

イベントの対応付けには `toolCallId` を使います。`tool_execution_update` の `partialResult` には差分ではなく、それまでに蓄積された出力全体が入るため、クライアントは更新ごとに表示全体を置き換えるだけで扱えます。

### queue_update

保留中の steering キューまたは follow-up キューに変更があるたびに発行されます。

```json
{
  "type": "queue_update",
  "steering": ["Focus on error handling"],
  "followUp": ["After that, summarize the result"]
}
```

### compaction_start / compaction_end

手動・自動を問わず、compaction が実行されると発行されます。

```json
{"type": "compaction_start", "reason": "threshold"}
```

`reason` フィールドは `"manual"`、`"threshold"`、または `"overflow"` です。

```json
{
  "type": "compaction_end",
  "reason": "threshold",
  "result": {
    "summary": "Summary of conversation...",
    "firstKeptEntryId": "abc123",
    "tokensBefore": 150000,
    "details": {}
  },
  "aborted": false,
  "willRetry": false
}
```

`reason` が `"overflow"` で compaction に成功した場合、`willRetry` は `true` となり、エージェントは自動的にプロンプトを再試行します。

compaction が中断された場合、`result` は `null`、`aborted` は `true` です。

compaction が失敗した場合（例: API quota 超過）、`result` は `null`、`aborted` は `false` で、`errorMessage` にエラー内容が入ります。

### auto_retry_start / auto_retry_end

一時エラー（overloaded、rate limit、5xx）の後で自動再試行が発動すると発行されます。

```json
{
  "type": "auto_retry_start",
  "attempt": 1,
  "maxAttempts": 3,
  "delayMs": 2000,
  "errorMessage": "529 {\"type\":\"error\",\"error\":{\"type\":\"overloaded_error\",\"message\":\"Overloaded\"}}"
}
```

```json
{
  "type": "auto_retry_end",
  "success": true,
  "attempt": 2
}
```

最終的に失敗した場合（最大再試行回数超過）:
```json
{
  "type": "auto_retry_end",
  "success": false,
  "attempt": 3,
  "finalError": "529 overloaded_error: Overloaded"
}
```

### extension_error

拡張がエラーを投げたときに発行されます。

```json
{
  "type": "extension_error",
  "extensionPath": "/path/to/extension.ts",
  "event": "tool_call",
  "error": "Error message..."
}
```

## Extension UI Protocol

拡張は `ctx.ui.select()`、`ctx.ui.confirm()` などを使ってユーザー操作を要求できます。RPC mode では、これらは基本の command/event フローの上に構築されたリクエスト/レスポンス型サブプロトコルへ変換されます。

拡張 UI メソッドには 2 つのカテゴリがあります。

- **Dialog method**（`select`、`confirm`、`input`、`editor`）: stdout に `extension_ui_request` を出力し、クライアントが一致する `id` を持つ `extension_ui_response` を stdin へ返すまで待機します。
- **Fire-and-forget method**（`notify`、`setStatus`、`setWidget`、`setTitle`、`set_editor_text`）: stdout に `extension_ui_request` を出力しますが、レスポンスは期待しません。クライアントは情報を表示しても無視しても構いません。

Dialog method に `timeout` フィールドがある場合、タイムアウト時にエージェント側がデフォルト値で自動解決します。クライアント側でタイムアウト管理を行う必要はありません。

一部の `ExtensionUIContext` メソッドは、TUI への直接アクセスを必要とするため、RPC mode では未対応または機能劣化します。
- `custom()` は `undefined` を返します
- `setWorkingMessage()`、`setWorkingIndicator()`、`setFooter()`、`setHeader()`、`setEditorComponent()`、`setToolsExpanded()` は no-op です
- `getEditorText()` は `""` を返します
- `getToolsExpanded()` は `false` を返します
- `pasteToEditor()` は `setEditorText()` へ委譲されます（paste/collapse の扱いはありません）
- `getAllThemes()` は `[]` を返します
- `getTheme()` は `undefined` を返します
- `setTheme()` は `{ success: false, error: "..." }` を返します

注: `ctx.hasUI` は、dialog と fire-and-forget の各メソッドが拡張 UI サブプロトコル経由で機能するため、RPC mode でも `true` です。

### Extension UI Requests (stdout)

すべてのリクエストは `type: "extension_ui_request"`、一意な `id`、および `method` フィールドを持ちます。

#### select

一覧からユーザーに選択させます。`timeout` フィールドを持つ dialog method は、ミリ秒単位のタイムアウトを含みます。クライアントが期限までに応答しない場合、エージェントは `undefined` で自動解決します。

```json
{
  "type": "extension_ui_request",
  "id": "uuid-1",
  "method": "select",
  "title": "Allow dangerous command?",
  "options": ["Allow", "Block"],
  "timeout": 10000
}
```

期待される応答: `value`（選択された文字列）または `cancelled: true` を持つ `extension_ui_response`。

#### confirm

yes/no の確認をユーザーに求めます。

```json
{
  "type": "extension_ui_request",
  "id": "uuid-2",
  "method": "confirm",
  "title": "Clear session?",
  "message": "All messages will be lost.",
  "timeout": 5000
}
```

期待される応答: `confirmed: true/false` または `cancelled: true` を持つ `extension_ui_response`。

#### input

自由入力テキストをユーザーに求めます。

```json
{
  "type": "extension_ui_request",
  "id": "uuid-3",
  "method": "input",
  "title": "Enter a value",
  "placeholder": "type something..."
}
```

期待される応答: `value`（入力文字列）または `cancelled: true` を持つ `extension_ui_response`。

#### editor

任意の初期テキスト付きで、複数行エディタを開きます。

```json
{
  "type": "extension_ui_request",
  "id": "uuid-4",
  "method": "editor",
  "title": "Edit some text",
  "prefill": "Line 1\nLine 2\nLine 3"
}
```

期待される応答: `value`（編集後テキスト）または `cancelled: true` を持つ `extension_ui_response`。

#### notify

通知を表示します。fire-and-forget のため、レスポンスは不要です。

```json
{
  "type": "extension_ui_request",
  "id": "uuid-5",
  "method": "notify",
  "message": "Command blocked by user",
  "notifyType": "warning"
}
```

`notifyType` フィールドは `"info"`、`"warning"`、または `"error"` です。省略時は `"info"` になります。

#### setStatus

フッターまたはステータスバーの status entry を設定または消去します。fire-and-forget です。

```json
{
  "type": "extension_ui_request",
  "id": "uuid-6",
  "method": "setStatus",
  "statusKey": "my-ext",
  "statusText": "Turn 3 running..."
}
```

そのキーの status entry を消去するには、`statusText: undefined` を送るか、省略してください。

#### setWidget

エディタの上または下に表示される widget（複数行テキストブロック）を設定または消去します。fire-and-forget です。

```json
{
  "type": "extension_ui_request",
  "id": "uuid-7",
  "method": "setWidget",
  "widgetKey": "my-ext",
  "widgetLines": ["--- My Widget ---", "Line 1", "Line 2"],
  "widgetPlacement": "aboveEditor"
}
```

widget を消去するには `widgetLines: undefined` を送るか、省略してください。`widgetPlacement` フィールドは `"aboveEditor"`（デフォルト）または `"belowEditor"` です。RPC mode では文字列配列のみをサポートし、component factory は無視されます。

#### setTitle

端末ウィンドウまたはタブのタイトルを設定します。fire-and-forget です。

```json
{
  "type": "extension_ui_request",
  "id": "uuid-8",
  "method": "setTitle",
  "title": "pi - my project"
}
```

#### set_editor_text

入力エディタ内のテキストを設定します。fire-and-forget です。

```json
{
  "type": "extension_ui_request",
  "id": "uuid-9",
  "method": "set_editor_text",
  "text": "prefilled text for the user"
}
```

### Extension UI Responses (stdin)

レスポンスは dialog method（`select`、`confirm`、`input`、`editor`）に対してのみ送信します。`id` はリクエストと一致している必要があります。

#### Value response (select, input, editor)

```json
{"type": "extension_ui_response", "id": "uuid-1", "value": "Allow"}
```

#### Confirmation response (confirm)

```json
{"type": "extension_ui_response", "id": "uuid-2", "confirmed": true}
```

#### Cancellation response (any dialog)

任意の dialog method を閉じます。拡張側では `undefined`（select/input/editor の場合）または `false`（confirm の場合）として受け取ります。

```json
{"type": "extension_ui_response", "id": "uuid-3", "cancelled": true}
```

## Error Handling

失敗したコマンドは、`success: false` を持つレスポンスを返します。

```json
{
  "type": "response",
  "command": "set_model",
  "success": false,
  "error": "Model not found: invalid/model"
}
```

パースエラー:

```json
{
  "type": "response",
  "command": "parse",
  "success": false,
  "error": "Failed to parse command: Unexpected token..."
}
```

## Types

ソースファイル:
- [`packages/ai/src/types.ts`](../../ai/src/types.ts) - `Model`, `UserMessage`, `AssistantMessage`, `ToolResultMessage`
- [`packages/agent/src/types.ts`](../../agent/src/types.ts) - `AgentMessage`, `AgentEvent`
- [`src/core/messages.ts`](../src/core/messages.ts) - `BashExecutionMessage`
- [`src/modes/rpc/rpc-types.ts`](../src/modes/rpc/rpc-types.ts) - RPC command/response 型、extension UI request/response 型

### Model

```json
{
  "id": "claude-sonnet-4-20250514",
  "name": "Claude Sonnet 4",
  "api": "anthropic-messages",
  "provider": "anthropic",
  "baseUrl": "https://api.anthropic.com",
  "reasoning": true,
  "input": ["text", "image"],
  "contextWindow": 200000,
  "maxTokens": 16384,
  "cost": {
    "input": 3.0,
    "output": 15.0,
    "cacheRead": 0.3,
    "cacheWrite": 3.75
  }
}
```

### UserMessage

```json
{
  "role": "user",
  "content": "Hello!",
  "timestamp": 1733234567890,
  "attachments": []
}
```

`content` フィールドは、文字列または `TextContent` / `ImageContent` ブロックの配列を取れます。

### AssistantMessage

```json
{
  "role": "assistant",
  "content": [
    {"type": "text", "text": "Hello! How can I help?"},
    {"type": "thinking", "thinking": "User is greeting me..."},
    {"type": "toolCall", "id": "call_123", "name": "bash", "arguments": {"command": "ls"}}
  ],
  "api": "anthropic-messages",
  "provider": "anthropic",
  "model": "claude-sonnet-4-20250514",
  "usage": {
    "input": 100,
    "output": 50,
    "cacheRead": 0,
    "cacheWrite": 0,
    "cost": {"input": 0.0003, "output": 0.00075, "cacheRead": 0, "cacheWrite": 0, "total": 0.00105}
  },
  "stopReason": "stop",
  "timestamp": 1733234567890
}
```

Stop reason: `"stop"`, `"length"`, `"toolUse"`, `"error"`, `"aborted"`

### ToolResultMessage

```json
{
  "role": "toolResult",
  "toolCallId": "call_123",
  "toolName": "bash",
  "content": [{"type": "text", "text": "total 48\ndrwxr-xr-x ..."}],
  "isError": false,
  "timestamp": 1733234567890
}
```

### BashExecutionMessage

`bash` RPC command によって作成されるメッセージです（LLM の tool call ではありません）。

```json
{
  "role": "bashExecution",
  "command": "ls -la",
  "output": "total 48\ndrwxr-xr-x ...",
  "exitCode": 0,
  "cancelled": false,
  "truncated": false,
  "fullOutputPath": null,
  "timestamp": 1733234567890
}
```

### Attachment

```json
{
  "id": "img1",
  "type": "image",
  "fileName": "photo.jpg",
  "mimeType": "image/jpeg",
  "size": 102400,
  "content": "base64-encoded-data...",
  "extractedText": null,
  "preview": null
}
```

## Example: Basic Client (Python)

```python
import subprocess
import json

proc = subprocess.Popen(
    ["pi", "--mode", "rpc", "--no-session"],
    stdin=subprocess.PIPE,
    stdout=subprocess.PIPE,
    text=True
)

def send(cmd):
    proc.stdin.write(json.dumps(cmd) + "\n")
    proc.stdin.flush()

def read_events():
    for line in proc.stdout:
        yield json.loads(line)

# Send prompt
send({"type": "prompt", "message": "Hello!"})

# Process events
for event in read_events():
    if event.get("type") == "message_update":
        delta = event.get("assistantMessageEvent", {})
        if delta.get("type") == "text_delta":
            print(delta["delta"], end="", flush=True)
    
    if event.get("type") == "agent_end":
        print()
        break
```

## Example: Interactive Client (Node.js)

完全な対話例は [`test/rpc-example.ts`](../test/rpc-example.ts) を、型付きクライアント実装は [`src/modes/rpc/rpc-client.ts`](../src/modes/rpc/rpc-client.ts) を参照してください。

拡張 UI プロトコルを扱う完全な例については、[`examples/rpc-extension-ui.ts`](../examples/rpc-extension-ui.ts) と、それに対応する拡張 [`examples/extensions/rpc-demo.ts`](../examples/extensions/rpc-demo.ts) を参照してください。

```javascript
const { spawn } = require("child_process");
const { StringDecoder } = require("string_decoder");

const agent = spawn("pi", ["--mode", "rpc", "--no-session"]);

function attachJsonlReader(stream, onLine) {
    const decoder = new StringDecoder("utf8");
    let buffer = "";

    stream.on("data", (chunk) => {
        buffer += typeof chunk === "string" ? chunk : decoder.write(chunk);

        while (true) {
            const newlineIndex = buffer.indexOf("\n");
            if (newlineIndex === -1) break;

            let line = buffer.slice(0, newlineIndex);
            buffer = buffer.slice(newlineIndex + 1);
            if (line.endsWith("\r")) line = line.slice(0, -1);
            onLine(line);
        }
    });

    stream.on("end", () => {
        buffer += decoder.end();
        if (buffer.length > 0) {
            onLine(buffer.endsWith("\r") ? buffer.slice(0, -1) : buffer);
        }
    });
}

attachJsonlReader(agent.stdout, (line) => {
    const event = JSON.parse(line);

    if (event.type === "message_update") {
        const { assistantMessageEvent } = event;
        if (assistantMessageEvent.type === "text_delta") {
            process.stdout.write(assistantMessageEvent.delta);
        }
    }
});

// Send prompt
agent.stdin.write(JSON.stringify({ type: "prompt", message: "Hello" }) + "\n");

// Abort on Ctrl+C
process.on("SIGINT", () => {
    agent.stdin.write(JSON.stringify({ type: "abort" }) + "\n");
});
```
