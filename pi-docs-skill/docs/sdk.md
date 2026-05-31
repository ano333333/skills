# SDK

> 要約: SDK は、pi のエージェント機能へプログラムからアクセスするための公式インターフェースです。既存アプリへの組み込み、独自 UI の構築、自動化ワークフローへの統合を行えます。セッション管理、モデル設定、ツール、拡張、スキル、実行モードまで一通り扱えます。

SDK は、pi のエージェント機能に対するプログラム的なアクセスを提供します。これを使うと、pi を他のアプリケーションへ組み込んだり、カスタムインターフェースを構築したり、自動化ワークフローへ統合したりできます。

**ユースケース例:**
- カスタム UI（Web、デスクトップ、モバイル）を構築する
- 既存アプリケーションへエージェント機能を統合する
- エージェントの推論を使う自動化パイプラインを作成する
- サブエージェントを起動するカスタムツールを構築する
- エージェントの振る舞いをプログラムからテストする

最小構成から完全制御までの実動例は [examples/sdk/](../examples/sdk/) を参照してください。

## Quick Start

```typescript
import { AuthStorage, createAgentSession, ModelRegistry, SessionManager } from "@earendil-works/pi-coding-agent";

// 資格情報ストレージとモデルレジストリを設定
const authStorage = AuthStorage.create();
const modelRegistry = ModelRegistry.create(authStorage);

const { session } = await createAgentSession({
  sessionManager: SessionManager.inMemory(),
  authStorage,
  modelRegistry,
});

session.subscribe((event) => {
  if (event.type === "message_update" && event.assistantMessageEvent.type === "text_delta") {
    process.stdout.write(event.assistantMessageEvent.delta);
  }
});

await session.prompt("現在のディレクトリにはどんなファイルがありますか？");
```

## Installation

```bash
npm install @earendil-works/pi-coding-agent
```

SDK はメインパッケージに同梱されています。別個にインストールする必要はありません。

## Core Concepts

### createAgentSession()

単一の `AgentSession` を作成するメインのファクトリ関数です。

`createAgentSession()` は `ResourceLoader` を使って、拡張、スキル、プロンプトテンプレート、テーマ、コンテキストファイルを供給します。明示的に渡さない場合、標準の探索を行う `DefaultResourceLoader` が使われます。

```typescript
import { createAgentSession, SessionManager } from "@earendil-works/pi-coding-agent";

// 最小構成: DefaultResourceLoader によるデフォルト設定
const { session } = await createAgentSession();

// カスタム構成: 特定のオプションだけ上書き
const { session } = await createAgentSession({
  model: myModel,
  tools: ["read", "bash"],
  sessionManager: SessionManager.inMemory(),
});
```

### AgentSession

セッションは、エージェントのライフサイクル、メッセージ履歴、モデル状態、圧縮、イベントストリーミングを管理します。

```typescript
interface AgentSession {
  // プロンプトを送信して完了まで待つ
  prompt(text: string, options?: PromptOptions): Promise<void>;

  // ストリーミング中にメッセージをキューする
  steer(text: string): Promise<void>;
  followUp(text: string): Promise<void>;

  // イベントを購読する（解除関数を返す）
  subscribe(listener: (event: AgentSessionEvent) => void): () => void;

  // セッション情報
  sessionFile: string | undefined;
  sessionId: string;

  // モデル制御
  setModel(model: Model): Promise<void>;
  setThinkingLevel(level: ThinkingLevel): void;
  cycleModel(): Promise<ModelCycleResult | undefined>;
  cycleThinkingLevel(): ThinkingLevel | undefined;

  // 状態アクセス
  agent: Agent;
  model: Model | undefined;
  thinkingLevel: ThinkingLevel;
  messages: AgentMessage[];
  isStreaming: boolean;

  // 現在のセッションファイル内でツリーをその場で移動する
  navigateTree(targetId: string, options?: { summarize?: boolean; customInstructions?: string; replaceInstructions?: boolean; label?: string }): Promise<{ editorText?: string; cancelled: boolean }>;

  // 圧縮
  compact(customInstructions?: string): Promise<CompactionResult>;
  abortCompaction(): void;

  // 現在の処理を中断
  abort(): Promise<void>;

  // 後始末
  dispose(): void;
}
```

new-session、resume、fork、import のようなセッション置換 API は `AgentSession` ではなく `AgentSessionRuntime` にあります。

### createAgentSessionRuntime() and AgentSessionRuntime

アクティブなセッションを差し替え、cwd に束縛されたランタイム状態を再構築する必要がある場合は、runtime API を使ってください。
これは組み込みの interactive、print、RPC モードでも使われている同じレイヤーです。

`createAgentSessionRuntime()` は runtime ファクトリと初期の cwd / セッション対象を受け取ります。ファクトリは process 全体で固定の入力を閉じ込め、有効な cwd に対して cwd 束縛サービスを再生成し、それらに対してセッションオプションを解決し、完全な runtime 結果を返します。

```typescript
import {
  type CreateAgentSessionRuntimeFactory,
  createAgentSessionFromServices,
  createAgentSessionRuntime,
  createAgentSessionServices,
  getAgentDir,
  SessionManager,
} from "@earendil-works/pi-coding-agent";

const createRuntime: CreateAgentSessionRuntimeFactory = async ({ cwd, sessionManager, sessionStartEvent }) => {
  const services = await createAgentSessionServices({ cwd });
  return {
    ...(await createAgentSessionFromServices({
      services,
      sessionManager,
      sessionStartEvent,
    })),
    services,
    diagnostics: services.diagnostics,
  };
};

const runtime = await createAgentSessionRuntime(createRuntime, {
  cwd: process.cwd(),
  agentDir: getAgentDir(),
  sessionManager: SessionManager.create(process.cwd()),
});
```

`AgentSessionRuntime` は、以下の処理にまたがってアクティブな runtime の置換を担います。

- `newSession()`
- `switchSession()`
- `fork()`
- `fork(entryId, { position: "at" })` による clone フロー
- `importFromJsonl()`

重要な挙動:

- これらの操作後、`runtime.session` は変化します
- イベント購読は特定の `AgentSession` に結び付いているため、置換後は再購読が必要です
- 拡張を使う場合、新しいセッションに対して `runtime.session.bindExtensions(...)` を再度呼び出してください
- 作成時の診断情報は `runtime.diagnostics` に返されます
- runtime の作成または置換に失敗するとメソッドは例外を投げ、呼び出し側が処理方針を決めます

```typescript
let session = runtime.session;
let unsubscribe = session.subscribe(() => {});

await runtime.newSession();

unsubscribe();
session = runtime.session;
unsubscribe = session.subscribe(() => {});
```

### Prompting and Message Queueing

`PromptOptions` は、プロンプト展開、ストリーミング中のキュー挙動、プロンプト事前検査通知を制御します。

```typescript
interface PromptOptions {
  expandPromptTemplates?: boolean;
  images?: ImageContent[];
  streamingBehavior?: "steer" | "followUp";
  source?: InputSource;
  preflightResult?: (success: boolean) => void;
}
```

`preflightResult` は `prompt()` の呼び出しごとに一度だけ呼ばれます。

- `true`: プロンプトが受理された、キューされた、または即時処理された
- `false`: 受理前の事前検査で拒否された

これは `prompt()` が解決する前に発火します。`prompt()` 自体は、再試行を含む受理済みの完全な実行が終わるまで解決しません。受理後の失敗は `preflightResult(false)` ではなく、通常のイベントとメッセージストリームで報告されます。

`prompt()` メソッドは、プロンプトテンプレート、拡張コマンド、メッセージ送信を扱います。

```typescript
// 基本的なプロンプト送信（ストリーミング中でない場合）
await session.prompt("ここにはどんなファイルがありますか？");

// 画像付き
await session.prompt("この画像には何が写っていますか？", {
  images: [{ type: "image", source: { type: "base64", mediaType: "image/png", data: "..." } }]
});

// ストリーミング中: メッセージのキュー方法を指定する必要がある
await session.prompt("止めて代わりにこれをしてください", { streamingBehavior: "steer" });
await session.prompt("終わったら X も確認してください", { streamingBehavior: "followUp" });
```

**挙動:**
- **拡張コマンド**（例: `/mycommand`）: ストリーミング中でも即座に実行されます。LLM とのやり取りは `pi.sendMessage()` を通じて自身で管理します。
- **ファイルベースのプロンプトテンプレート**（`.md` ファイル由来）: 送信またはキュー前にその内容へ展開されます。
- **`streamingBehavior` なしでストリーミング中に呼ぶ**: エラーになります。`steer()` / `followUp()` を直接使うか、このオプションを指定してください。
- **`preflightResult(true)`**: プロンプトが受理された、キューされた、または即時処理されたことを意味します。
- **`preflightResult(false)`**: 受理前の事前検査で拒否されたことを意味します。

ストリーミング中に明示的にキューするには:

```typescript
// 現在のアシスタントターンのツール呼び出しが終わった後に配信される指示メッセージをキューする
await session.steer("新しい指示");

// エージェントの終了を待ってから配信される
await session.followUp("終わったら、これもやってください");
```

`steer()` と `followUp()` はどちらもファイルベースのプロンプトテンプレートを展開しますが、拡張コマンドに対してはエラーになります（拡張コマンドはキューできません）。

### Agent and AgentState

`Agent` クラス（`@earendil-works/pi-agent-core` 由来）は、LLM との中核的なやり取りを処理します。`session.agent` からアクセスできます。

```typescript
// 現在の状態にアクセス
const state = session.agent.state;

// state.messages: AgentMessage[] - 会話履歴
// state.model: Model - 現在のモデル
// state.thinkingLevel: ThinkingLevel - 現在の thinking レベル
// state.systemPrompt: string - システムプロンプト
// state.tools: AgentTool[] - 利用可能なツール
// state.streamingMessage?: AgentMessage - 現在の部分的なアシスタントメッセージ
// state.errorMessage?: string - 直近のアシスタントエラー

// メッセージを置き換える（分岐や復元に便利）
session.agent.state.messages = messages; // トップレベル配列をコピーする

// ツールを置き換える
session.agent.state.tools = tools; // トップレベル配列をコピーする

// エージェントの処理完了を待つ
await session.agent.waitForIdle();
```

### Events

イベントを購読すると、ストリーミング出力やライフサイクル通知を受け取れます。

```typescript
session.subscribe((event) => {
  switch (event.type) {
    // アシスタントからのストリーミングテキスト
    case "message_update":
      if (event.assistantMessageEvent.type === "text_delta") {
        process.stdout.write(event.assistantMessageEvent.delta);
      }
      if (event.assistantMessageEvent.type === "thinking_delta") {
        // Thinking 出力（thinking が有効な場合）
      }
      break;

    // ツール実行
    case "tool_execution_start":
      console.log(`ツール: ${event.toolName}`);
      break;
    case "tool_execution_update":
      // ストリーミングされるツール出力
      break;
    case "tool_execution_end":
      console.log(`結果: ${event.isError ? "error" : "success"}`);
      break;

    // メッセージのライフサイクル
    case "message_start":
      // 新しいメッセージ開始
      break;
    case "message_end":
      // メッセージ完了
      break;

    // エージェントのライフサイクル
    case "agent_start":
      // エージェントがプロンプト処理を開始
      break;
    case "agent_end":
      // エージェントが完了（event.messages に新規メッセージが入る）
      break;

    // ターンのライフサイクル（1 回の LLM 応答 + ツール呼び出し）
    case "turn_start":
      break;
    case "turn_end":
      // event.message: アシスタントの応答
      // event.toolResults: このターンのツール結果
      break;

    // セッションイベント（キュー、圧縮、再試行）
    case "queue_update":
      console.log(event.steering, event.followUp);
      break;
    case "compaction_start":
    case "compaction_end":
    case "auto_retry_start":
    case "auto_retry_end":
      break;
  }
});
```

## Options Reference

### Directories

```typescript
const { session } = await createAgentSession({
  // DefaultResourceLoader が探索に使う作業ディレクトリ
  cwd: process.cwd(), // default

  // グローバル設定ディレクトリ
  agentDir: "~/.pi/agent", // default (~ を展開)
});
```

`cwd` は `DefaultResourceLoader` において以下に使われます。
- プロジェクト拡張（`.pi/extensions/`）
- プロジェクトスキル:
  - `.pi/skills/`
  - `cwd` とその祖先ディレクトリ（git リポジトリのルート、またはリポジトリ外ならファイルシステムルートまで）にある `.agents/skills/`
- プロジェクトプロンプト（`.pi/prompts/`）
- コンテキストファイル（`cwd` から上方向にたどる `AGENTS.md`）
- セッションディレクトリ命名

`agentDir` は `DefaultResourceLoader` において以下に使われます。
- グローバル拡張（`extensions/`）
- グローバルスキル:
  - `agentDir` 配下の `skills/`（例: `~/.pi/agent/skills/`）
  - `~/.agents/skills/`
- グローバルプロンプト（`prompts/`）
- グローバルコンテキストファイル（`AGENTS.md`）
- 設定（`settings.json`）
- カスタムモデル（`models.json`）
- 認証情報（`auth.json`）
- セッション（`sessions/`）

カスタム `ResourceLoader` を渡すと、`cwd` と `agentDir` はリソース探索を制御しなくなります。それでもセッション命名やツールのパス解決には影響します。

### Model

```typescript
import { getModel } from "@earendil-works/pi-ai";
import { AuthStorage, ModelRegistry } from "@earendil-works/pi-coding-agent";

const authStorage = AuthStorage.create();
const modelRegistry = ModelRegistry.create(authStorage);

// 特定の組み込みモデルを探す（API キーの存在は確認しない）
const opus = getModel("anthropic", "claude-opus-4-5");
if (!opus) throw new Error("Model not found");

// provider/id で任意のモデルを探す。models.json のカスタムモデルも含む
// （API キーの存在は確認しない）
const customModel = modelRegistry.find("my-provider", "my-model");

// 有効な API キーが設定されているモデルだけを取得
const available = await modelRegistry.getAvailable();

const { session } = await createAgentSession({
  model: opus,
  thinkingLevel: "medium", // off, minimal, low, medium, high, xhigh

  // モデル切り替え用（interactive モードでは Ctrl+P）
  scopedModels: [
    { model: opus, thinkingLevel: "high" },
    { model: haiku, thinkingLevel: "off" },
  ],

  authStorage,
  modelRegistry,
});
```

モデルを明示しない場合:
1. セッションから復元を試みる（継続時）
2. settings のデフォルトを使う
3. 最初に利用可能なモデルへフォールバックする

> [examples/sdk/02-custom-model.ts](../examples/sdk/02-custom-model.ts) を参照してください

### API Keys and OAuth

API キー解決の優先順位（`AuthStorage` が処理）:
1. ランタイム上書き（`setRuntimeApiKey` 経由。永続化されない）
2. `auth.json` に保存された認証情報（API キーまたは OAuth トークン）
3. 環境変数（`ANTHROPIC_API_KEY`、`OPENAI_API_KEY` など）
4. フォールバックリゾルバ（`models.json` にあるカスタムプロバイダキー用）

```typescript
import { AuthStorage, ModelRegistry } from "@earendil-works/pi-coding-agent";

// デフォルト: ~/.pi/agent/auth.json と ~/.pi/agent/models.json を使う
const authStorage = AuthStorage.create();
const modelRegistry = ModelRegistry.create(authStorage);

const { session } = await createAgentSession({
  sessionManager: SessionManager.inMemory(),
  authStorage,
  modelRegistry,
});

// ランタイム API キー上書き（ディスクへ永続化されない）
authStorage.setRuntimeApiKey("anthropic", "sk-my-temp-key");

// カスタム認証ストレージの場所
const customAuth = AuthStorage.create("/my/app/auth.json");
const customRegistry = ModelRegistry.create(customAuth, "/my/app/models.json");

const { session } = await createAgentSession({
  sessionManager: SessionManager.inMemory(),
  authStorage: customAuth,
  modelRegistry: customRegistry,
});

// カスタム models.json なし（組み込みモデルのみ）
const simpleRegistry = ModelRegistry.inMemory(authStorage);
```

> [examples/sdk/09-api-keys-and-oauth.ts](../examples/sdk/09-api-keys-and-oauth.ts) を参照してください

### System Prompt

システムプロンプトを上書きするには `ResourceLoader` を使います。

```typescript
import { createAgentSession, DefaultResourceLoader } from "@earendil-works/pi-coding-agent";

const loader = new DefaultResourceLoader({
  systemPromptOverride: () => "あなたは有能なアシスタントです。",
});
await loader.reload();

const { session } = await createAgentSession({ resourceLoader: loader });
```

> [examples/sdk/03-custom-prompt.ts](../examples/sdk/03-custom-prompt.ts) を参照してください

### Tools

有効化する組み込みツールを指定します。

- 組み込みツール名: `read`, `bash`, `edit`, `write`, `grep`, `find`, `ls`
- 既定の組み込み: `read`, `bash`, `edit`, `write`
- `noTools: "all"` はすべてのツールを無効化します
- `noTools: "builtin"` は既定の組み込みツールを無効化しつつ、拡張ツールとカスタムツールは有効のままにします
- `excludeTools` は、`tools` の許可リスト適用後に特定の組み込み、拡張、カスタムツール名を無効化します

`edit` ツールは、Pi の TUI 表示向けに `details.diff` を返し、SDK 利用者向けに標準的な unified patch として `details.patch` も返します。

```typescript
import { createAgentSession } from "@earendil-works/pi-coding-agent";

// 読み取り専用モード
const { session } = await createAgentSession({
  tools: ["read", "grep", "find", "ls"],
});

// 特定のツールだけを選択
const { session } = await createAgentSession({
  tools: ["read", "bash", "grep"],
});

// 1 つのツールだけ無効化し、残りは利用可能なままにする
const { session } = await createAgentSession({
  excludeTools: ["ask_question"],
});
```

#### Tools with Custom cwd

カスタム `cwd` を渡すと、`createAgentSession()` はその cwd 向けに選択された組み込みツールを構築します。

```typescript
import { createAgentSession, SessionManager } from "@earendil-works/pi-coding-agent";

const cwd = "/path/to/project";

// カスタム cwd に対して既定ツールを使う
const { session } = await createAgentSession({
  cwd,
  sessionManager: SessionManager.inMemory(cwd),
});

// またはカスタム cwd に対して特定のツールだけを使う
const { session } = await createAgentSession({
  cwd,
  tools: ["read", "bash", "grep"],
  sessionManager: SessionManager.inMemory(cwd),
});
```

> [examples/sdk/05-tools.ts](../examples/sdk/05-tools.ts) を参照してください

### Custom Tools

```typescript
import { Type } from "typebox";
import { createAgentSession, defineTool } from "@earendil-works/pi-coding-agent";

// インラインのカスタムツール
const myTool = defineTool({
  name: "my_tool",
  label: "My Tool",
  description: "役に立つ処理を行う",
  parameters: Type.Object({
    input: Type.String({ description: "入力値" }),
  }),
  execute: async (_toolCallId, params) => ({
    content: [{ type: "text", text: `結果: ${params.input}` }],
    details: {},
  }),
});

// カスタムツールを直接渡す
const { session } = await createAgentSession({
  customTools: [myTool],
});
```

独立した定義には `defineTool()` を使い、`customTools: [myTool]` のような配列で渡します。インラインの `pi.registerTool({ ... })` はすでにパラメータ型を正しく推論します。

`customTools` 経由で渡したカスタムツールは、拡張が登録したツールと結合されます。`ResourceLoader` に読み込まれた拡張も、`pi.registerTool()` 経由でツールを登録できます。

`tools` を渡す場合、`tools: ["read", "bash", "my_tool"]` のように、有効化したい各カスタムツール名または拡張ツール名を含めてください。

> [examples/sdk/05-tools.ts](../examples/sdk/05-tools.ts) を参照してください

### Extensions

拡張は `ResourceLoader` によって読み込まれます。`DefaultResourceLoader` は `~/.pi/agent/extensions/`、`.pi/extensions/`、および settings.json の extension sources から拡張を探索します。

```typescript
import { createAgentSession, DefaultResourceLoader } from "@earendil-works/pi-coding-agent";

const loader = new DefaultResourceLoader({
  additionalExtensionPaths: ["/path/to/my-extension.ts"],
  extensionFactories: [
    (pi) => {
      pi.on("agent_start", () => {
        console.log("[Inline Extension] エージェント開始");
      });
    },
  ],
});
await loader.reload();

const { session } = await createAgentSession({ resourceLoader: loader });
```

拡張は、ツールの登録、イベント購読、コマンド追加などを行えます。完全な API は [extensions.md](extensions.md) を参照してください。

**Event Bus:** 拡張は `pi.events` を通じて相互通信できます。外部から emit / listen したい場合は、共有 `eventBus` を `DefaultResourceLoader` に渡してください。

```typescript
import { createEventBus, DefaultResourceLoader } from "@earendil-works/pi-coding-agent";

const eventBus = createEventBus();
const loader = new DefaultResourceLoader({
  eventBus,
});
await loader.reload();

eventBus.on("my-extension:status", (data) => console.log(data));
```

> [examples/sdk/06-extensions.ts](../examples/sdk/06-extensions.ts) と [docs/extensions.md](extensions.md) を参照してください

### Skills

```typescript
import {
  createAgentSession,
  DefaultResourceLoader,
  type Skill,
} from "@earendil-works/pi-coding-agent";

const customSkill: Skill = {
  name: "my-skill",
  description: "カスタム指示",
  filePath: "/path/to/SKILL.md",
  baseDir: "/path/to",
  source: "custom",
};

const loader = new DefaultResourceLoader({
  skillsOverride: (current) => ({
    skills: [...current.skills, customSkill],
    diagnostics: current.diagnostics,
  }),
});
await loader.reload();

const { session } = await createAgentSession({ resourceLoader: loader });
```

> [examples/sdk/04-skills.ts](../examples/sdk/04-skills.ts) を参照してください

### Context Files

```typescript
import { createAgentSession, DefaultResourceLoader } from "@earendil-works/pi-coding-agent";

const loader = new DefaultResourceLoader({
  agentsFilesOverride: (current) => ({
    agentsFiles: [
      ...current.agentsFiles,
      { path: "/virtual/AGENTS.md", content: "# ガイドライン\n\n- 簡潔に答える" },
    ],
  }),
});
await loader.reload();

const { session } = await createAgentSession({ resourceLoader: loader });
```

> [examples/sdk/07-context-files.ts](../examples/sdk/07-context-files.ts) を参照してください

### Slash Commands

```typescript
import {
  createAgentSession,
  DefaultResourceLoader,
  type PromptTemplate,
} from "@earendil-works/pi-coding-agent";

const customCommand: PromptTemplate = {
  name: "deploy",
  description: "アプリケーションをデプロイする",
  source: "(custom)",
  content: "# Deploy\n\n1. Build\n2. Test\n3. Deploy",
};

const loader = new DefaultResourceLoader({
  promptsOverride: (current) => ({
    prompts: [...current.prompts, customCommand],
    diagnostics: current.diagnostics,
  }),
});
await loader.reload();

const { session } = await createAgentSession({ resourceLoader: loader });
```

> [examples/sdk/08-prompt-templates.ts](../examples/sdk/08-prompt-templates.ts) を参照してください

### Session Management

セッションは `id` / `parentId` で結び付いたツリー構造を使い、その場での分岐を可能にします。

```typescript
import {
  type CreateAgentSessionRuntimeFactory,
  createAgentSession,
  createAgentSessionFromServices,
  createAgentSessionRuntime,
  createAgentSessionServices,
  getAgentDir,
  SessionManager,
} from "@earendil-works/pi-coding-agent";

// インメモリ（永続化なし）
const { session } = await createAgentSession({
  sessionManager: SessionManager.inMemory(),
});

// 新しい永続セッション
const { session: persisted } = await createAgentSession({
  sessionManager: SessionManager.create(process.cwd()),
});

// 最新を継続
const { session: continued, modelFallbackMessage } = await createAgentSession({
  sessionManager: SessionManager.continueRecent(process.cwd()),
});
if (modelFallbackMessage) {
  console.log("注意:", modelFallbackMessage);
}

// 特定ファイルを開く
const { session: opened } = await createAgentSession({
  sessionManager: SessionManager.open("/path/to/session.jsonl"),
});

// セッション一覧
const currentProjectSessions = await SessionManager.list(process.cwd());
const allSessions = await SessionManager.listAll(process.cwd());

// /new, /resume, /fork, /clone, import フロー向けのセッション置換 API
const createRuntime: CreateAgentSessionRuntimeFactory = async ({ cwd, sessionManager, sessionStartEvent }) => {
  const services = await createAgentSessionServices({ cwd });
  return {
    ...(await createAgentSessionFromServices({
      services,
      sessionManager,
      sessionStartEvent,
    })),
    services,
    diagnostics: services.diagnostics,
  };
};

const runtime = await createAgentSessionRuntime(createRuntime, {
  cwd: process.cwd(),
  agentDir: getAgentDir(),
  sessionManager: SessionManager.create(process.cwd()),
});

// アクティブなセッションを新しいものへ置き換える
await runtime.newSession();

// アクティブなセッションを別の保存済みセッションへ置き換える
await runtime.switchSession("/path/to/session.jsonl");

// 特定のユーザーエントリから fork したセッションへ置き換える
await runtime.fork("entry-id");

// 特定エントリを通るアクティブパスを clone する
await runtime.fork("entry-id", { position: "at" });
```

**SessionManager tree API:**

```typescript
const sm = SessionManager.open("/path/to/session.jsonl");

// セッション一覧
const currentProjectSessions = await SessionManager.list(process.cwd());
const allSessions = await SessionManager.listAll(process.cwd());

// ツリー走査
const entries = sm.getEntries();        // すべてのエントリ（ヘッダ除く）
const tree = sm.getTree();              // 完全なツリー構造
const path = sm.getPath();              // ルートから現在の葉までのパス
const leaf = sm.getLeafEntry();         // 現在の葉エントリ
const entry = sm.getEntry(id);          // ID でエントリ取得
const children = sm.getChildren(id);    // エントリの直接の子

// ラベル
const label = sm.getLabel(id);          // エントリのラベル取得
sm.appendLabelChange(id, "checkpoint"); // ラベル設定

// 分岐
sm.branch(entryId);                      // 葉を過去のエントリへ移動
sm.branchWithSummary(id, "要約...");     // コンテキスト要約付きで分岐
sm.createBranchedSession(leafId);        // パスを新しいファイルへ抽出
```

> [examples/sdk/11-sessions.ts](../examples/sdk/11-sessions.ts) と [Session Format](session-format.md) を参照してください

### Settings Management

```typescript
import { createAgentSession, SettingsManager, SessionManager } from "@earendil-works/pi-coding-agent";

// デフォルト: ファイルから読み込む（グローバル + プロジェクトをマージ）
const { session } = await createAgentSession({
  settingsManager: SettingsManager.create(),
});

// 上書きあり
const settingsManager = SettingsManager.create();
settingsManager.applyOverrides({
  compaction: { enabled: false },
  retry: { enabled: true, maxRetries: 5 },
});
const { session } = await createAgentSession({ settingsManager });

// インメモリ（ファイル I/O なし。テスト向け）
const { session } = await createAgentSession({
  settingsManager: SettingsManager.inMemory({ compaction: { enabled: false } }),
  sessionManager: SessionManager.inMemory(),
});

// カスタムディレクトリ
const { session } = await createAgentSession({
  settingsManager: SettingsManager.create("/custom/cwd", "/custom/agent"),
});
```

**静的ファクトリ:**
- `SettingsManager.create(cwd?, agentDir?)` - ファイルから読み込む
- `SettingsManager.inMemory(settings?)` - ファイル I/O なし

**プロジェクト固有設定:**

設定は 2 か所から読み込まれてマージされます。
1. グローバル: `~/.pi/agent/settings.json`
2. プロジェクト: `<cwd>/.pi/settings.json`

プロジェクト設定がグローバルを上書きします。ネストしたオブジェクトはキー単位でマージされます。setter はデフォルトでグローバル設定を変更します。

**永続化とエラー処理の意味論:**

- 設定 getter / setter は、インメモリ状態に対して同期的です。
- setter は非同期に永続化書き込みをキューします。
- プロセス終了前や、テストでファイル内容を検証する前など、永続化境界が必要なら `await settingsManager.flush()` を呼んでください。
- `SettingsManager` 自体は設定 I/O エラーを出力しません。`settingsManager.drainErrors()` を使い、アプリ側レイヤーで報告してください。

> [examples/sdk/10-settings.ts](../examples/sdk/10-settings.ts) を参照してください

## ResourceLoader

`DefaultResourceLoader` を使うと、拡張、スキル、プロンプト、テーマ、コンテキストファイルを探索できます。

```typescript
import {
  DefaultResourceLoader,
  getAgentDir,
} from "@earendil-works/pi-coding-agent";

const loader = new DefaultResourceLoader({
  cwd,
  agentDir: getAgentDir(),
});
await loader.reload();

const extensions = loader.getExtensions();
const skills = loader.getSkills();
const prompts = loader.getPrompts();
const themes = loader.getThemes();
const contextFiles = loader.getAgentsFiles().agentsFiles;
```

## Return Value

`createAgentSession()` の戻り値:

```typescript
interface CreateAgentSessionResult {
  // セッション本体
  session: AgentSession;

  // 拡張の結果（ランナー設定用）
  extensionsResult: LoadExtensionsResult;

  // セッションのモデルを復元できなかった場合の警告
  modelFallbackMessage?: string;
}

interface LoadExtensionsResult {
  extensions: Extension[];
  errors: Array<{ path: string; error: string }>;
  runtime: ExtensionRuntime;
}
```

## Complete Example

```typescript
import { getModel } from "@earendil-works/pi-ai";
import { Type } from "typebox";
import {
  AuthStorage,
  createAgentSession,
  DefaultResourceLoader,
  defineTool,
  ModelRegistry,
  SessionManager,
  SettingsManager,
} from "@earendil-works/pi-coding-agent";

// 認証ストレージを設定（カスタム保存先）
const authStorage = AuthStorage.create("/custom/agent/auth.json");

// ランタイム API キー上書き（永続化されない）
if (process.env.MY_KEY) {
  authStorage.setRuntimeApiKey("anthropic", process.env.MY_KEY);
}

// モデルレジストリ（カスタム models.json なし）
const modelRegistry = ModelRegistry.create(authStorage);

// インラインツール
const statusTool = defineTool({
  name: "status",
  label: "Status",
  description: "システム状態を取得する",
  parameters: Type.Object({}),
  execute: async () => ({
    content: [{ type: "text", text: `Uptime: ${process.uptime()}s` }],
    details: {},
  }),
});

const model = getModel("anthropic", "claude-opus-4-5");
if (!model) throw new Error("Model not found");

// 上書き付きのインメモリ設定
const settingsManager = SettingsManager.inMemory({
  compaction: { enabled: false },
  retry: { enabled: true, maxRetries: 2 },
});

const loader = new DefaultResourceLoader({
  cwd: process.cwd(),
  agentDir: "/custom/agent",
  settingsManager,
  systemPromptOverride: () => "あなたは最小限の応答をするアシスタントです。簡潔に答えてください。",
});
await loader.reload();

const { session } = await createAgentSession({
  cwd: process.cwd(),
  agentDir: "/custom/agent",

  model,
  thinkingLevel: "off",
  authStorage,
  modelRegistry,

  tools: ["read", "bash", "status"],
  customTools: [statusTool],
  resourceLoader: loader,

  sessionManager: SessionManager.inMemory(),
  settingsManager,
});

session.subscribe((event) => {
  if (event.type === "message_update" && event.assistantMessageEvent.type === "text_delta") {
    process.stdout.write(event.assistantMessageEvent.delta);
  }
});

await session.prompt("状態を取得してファイル一覧を表示してください。");
```

## Run Modes

SDK は、`createAgentSession()` の上にカスタムインターフェースを構築するための実行モード用ユーティリティをエクスポートしています。

### InteractiveMode

エディタ、チャット履歴、すべての組み込みコマンドを備えた完全な TUI インタラクティブモードです。

```typescript
import {
  type CreateAgentSessionRuntimeFactory,
  createAgentSessionFromServices,
  createAgentSessionRuntime,
  createAgentSessionServices,
  getAgentDir,
  InteractiveMode,
  SessionManager,
} from "@earendil-works/pi-coding-agent";

const createRuntime: CreateAgentSessionRuntimeFactory = async ({ cwd, sessionManager, sessionStartEvent }) => {
  const services = await createAgentSessionServices({ cwd });
  return {
    ...(await createAgentSessionFromServices({ services, sessionManager, sessionStartEvent })),
    services,
    diagnostics: services.diagnostics,
  };
};
const runtime = await createAgentSessionRuntime(createRuntime, {
  cwd: process.cwd(),
  agentDir: getAgentDir(),
  sessionManager: SessionManager.create(process.cwd()),
});

const mode = new InteractiveMode(runtime, {
  migratedProviders: [],
  modelFallbackMessage: undefined,
  initialMessage: "こんにちは",
  initialImages: [],
  initialMessages: [],
});

await mode.run();
```

### runPrintMode

単発実行モードです。プロンプトを送り、結果を出力して、終了します。

```typescript
import {
  type CreateAgentSessionRuntimeFactory,
  createAgentSessionFromServices,
  createAgentSessionRuntime,
  createAgentSessionServices,
  getAgentDir,
  runPrintMode,
  SessionManager,
} from "@earendil-works/pi-coding-agent";

const createRuntime: CreateAgentSessionRuntimeFactory = async ({ cwd, sessionManager, sessionStartEvent }) => {
  const services = await createAgentSessionServices({ cwd });
  return {
    ...(await createAgentSessionFromServices({ services, sessionManager, sessionStartEvent })),
    services,
    diagnostics: services.diagnostics,
  };
};
const runtime = await createAgentSessionRuntime(createRuntime, {
  cwd: process.cwd(),
  agentDir: getAgentDir(),
  sessionManager: SessionManager.create(process.cwd()),
});

await runPrintMode(runtime, {
  mode: "text",
  initialMessage: "こんにちは",
  initialImages: [],
  messages: ["続けて確認してください"],
});
```

### runRpcMode

サブプロセス統合向けの JSON-RPC モードです。

```typescript
import {
  type CreateAgentSessionRuntimeFactory,
  createAgentSessionFromServices,
  createAgentSessionRuntime,
  createAgentSessionServices,
  getAgentDir,
  runRpcMode,
  SessionManager,
} from "@earendil-works/pi-coding-agent";

const createRuntime: CreateAgentSessionRuntimeFactory = async ({ cwd, sessionManager, sessionStartEvent }) => {
  const services = await createAgentSessionServices({ cwd });
  return {
    ...(await createAgentSessionFromServices({ services, sessionManager, sessionStartEvent })),
    services,
    diagnostics: services.diagnostics,
  };
};
const runtime = await createAgentSessionRuntime(createRuntime, {
  cwd: process.cwd(),
  agentDir: getAgentDir(),
  sessionManager: SessionManager.create(process.cwd()),
});

await runRpcMode(runtime);
```

JSON プロトコルについては [RPC documentation](rpc.md) を参照してください。

## RPC Mode Alternative

SDK を使って組み込まず、サブプロセスベースで統合したい場合は CLI を直接使う方法もあります。

```bash
pi --mode rpc --no-session
```

JSON プロトコルについては [RPC documentation](rpc.md) を参照してください。

SDK が適しているケース:
- 型安全性が欲しい
- 同じ Node.js プロセス内で動かす
- エージェント状態へ直接アクセスしたい
- ツールや拡張をプログラムからカスタマイズしたい

RPC モードが適しているケース:
- 他言語から統合したい
- プロセス分離が欲しい
- 言語非依存のクライアントを構築したい

## Exports

メインエントリポイントは以下をエクスポートします。

```typescript
// ファクトリ
createAgentSession
createAgentSessionRuntime
AgentSessionRuntime

// 認証とモデル
AuthStorage
ModelRegistry

// リソース読み込み
DefaultResourceLoader
type ResourceLoader
createEventBus

// ヘルパー
defineTool

// セッション管理
SessionManager
SettingsManager

// ツールファクトリ
createCodingTools
createReadOnlyTools
createReadTool, createBashTool, createEditTool, createWriteTool
createGrepTool, createFindTool, createLsTool

// 型
type CreateAgentSessionOptions
type CreateAgentSessionResult
type ExtensionFactory
type ExtensionAPI
type ToolDefinition
type Skill
type PromptTemplate
type Tool
```

拡張用の型については、完全な API を [extensions.md](extensions.md) で確認してください。
