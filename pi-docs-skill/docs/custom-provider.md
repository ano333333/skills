# カスタムプロバイダー

> 要約: 拡張機能は `pi.registerProvider()` を通じて、既存プロバイダーの上書きや新規プロバイダーの登録を行えます。プロキシ経由の接続、独自エンドポイント、OAuth/SSO、非標準 API 向けのストリーミング実装まで扱えるため、企業内環境や独自基盤への統合に向いています。

拡張機能は `pi.registerProvider()` によってカスタムモデルプロバイダーを登録できます。これにより、以下が可能になります。

- **プロキシ** - リクエストを企業プロキシや API ゲートウェイ経由でルーティングする
- **カスタムエンドポイント** - セルフホストまたはプライベートなモデルデプロイを利用する
- **OAuth/SSO** - 企業向けプロバイダーの認証フローを追加する
- **カスタム API** - 非標準 LLM API 向けのストリーミングを実装する

## 拡張機能の例

完全なプロバイダー実装例は以下を参照してください。

- [`examples/extensions/custom-provider-anthropic/`](../examples/extensions/custom-provider-anthropic/)
- [`examples/extensions/custom-provider-gitlab-duo/`](../examples/extensions/custom-provider-gitlab-duo/)

## 目次

- [拡張機能の例](#拡張機能の例)
- [クイックリファレンス](#クイックリファレンス)
- [既存プロバイダーを上書きする](#既存プロバイダーを上書きする)
- [新しいプロバイダーを登録する](#新しいプロバイダーを登録する)
- [プロバイダー登録を解除する](#プロバイダー登録を解除する)
- [OAuth サポート](#oauth-サポート)
- [カスタムストリーミング API](#カスタムストリーミング-api)
- [コンテキストあふれエラー](#コンテキストあふれエラー)
- [実装のテスト](#実装のテスト)
- [設定リファレンス](#設定リファレンス)
- [モデル定義リファレンス](#モデル定義リファレンス)

## クイックリファレンス

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";

export default function (pi: ExtensionAPI) {
  // 既存プロバイダーの baseUrl を上書き
  pi.registerProvider("anthropic", {
    baseUrl: "https://proxy.example.com"
  });

  // モデル付きで新規プロバイダーを登録
  pi.registerProvider("my-provider", {
    name: "My Provider",
    baseUrl: "https://api.example.com",
    apiKey: "$MY_API_KEY",
    api: "openai-completions",
    models: [
      {
        id: "my-model",
        name: "My Model",
        reasoning: false,
        input: ["text", "image"],
        cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
        contextWindow: 128000,
        maxTokens: 4096
      }
    ]
  });
}
```

拡張ファクトリーは `async` にすることもできます。動的なモデル検出が必要な場合は、`session_start` ではなくファクトリー内でモデルを取得して登録してください。pi は起動処理を続ける前にファクトリーの完了を待つため、対話起動中や `pi --list-models` に対してそのプロバイダーが利用可能になります。

## 既存プロバイダーを上書きする

最も単純な用途は、既存のプロバイダーをプロキシ経由に切り替えることです。

```typescript
// 以後の Anthropic リクエストはすべてあなたのプロキシを経由する
pi.registerProvider("anthropic", {
  baseUrl: "https://proxy.example.com"
});

// OpenAI リクエストにカスタムヘッダーを追加する
pi.registerProvider("openai", {
  headers: {
    "X-Custom-Header": "value"
  }
});

// baseUrl と headers の両方を設定する
pi.registerProvider("google", {
  baseUrl: "https://ai-gateway.corp.com/google",
  headers: {
    "X-Corp-Auth": "$CORP_AUTH_TOKEN"  // 環境変数またはリテラル
  }
});
```

`baseUrl` と `headers` のみを指定した場合（`models` なし）、そのプロバイダーの既存モデルはすべて、新しいエンドポイント設定を付けたまま保持されます。

## 新しいプロバイダーを登録する

まったく新しいプロバイダーを追加するには、必要な設定とあわせて `models` を指定します。

モデル一覧をリモートエンドポイントから取得する場合は、非同期の拡張ファクトリーを使ってください。

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

これにより、取得したモデル群は起動完了前に登録されます。

```typescript
pi.registerProvider("my-llm", {
  baseUrl: "https://api.my-llm.com/v1",
  apiKey: "$MY_LLM_API_KEY",  // 環境変数参照
  api: "openai-completions",  // 使用するストリーミング API
  models: [
    {
      id: "my-llm-large",
      name: "My LLM Large",
      reasoning: true,        // 拡張思考をサポート
      input: ["text", "image"],
      cost: {
        input: 3.0,           // 100 万トークンあたりのドル価格
        output: 15.0,
        cacheRead: 0.3,
        cacheWrite: 3.75
      },
      contextWindow: 200000,
      maxTokens: 16384
    }
  ]
});
```

`models` を指定すると、そのプロバイダーに既存で登録されていたすべてのモデルは**置き換え**られます。

`apiKey` とカスタムヘッダー値は、`models.json` と同じ設定値構文を使います。先頭の `!command` は値全体をコマンド実行し、`$ENV_VAR` と `${ENV_VAR}` は環境変数を展開し、`$$` はリテラルの `$` を、`$!` はリテラルの `!` を出力します。

## プロバイダー登録を解除する

`pi.unregisterProvider(name)` を使うと、`pi.registerProvider(name, ...)` で以前に登録したプロバイダーを削除できます。

```typescript
// 登録
pi.registerProvider("my-llm", {
  baseUrl: "https://api.my-llm.com/v1",
  apiKey: "$MY_LLM_API_KEY",
  api: "openai-completions",
  models: [
    {
      id: "my-llm-large",
      name: "My LLM Large",
      reasoning: true,
      input: ["text", "image"],
      cost: { input: 3.0, output: 15.0, cacheRead: 0.3, cacheWrite: 3.75 },
      contextWindow: 200000,
      maxTokens: 16384
    }
  ]
});

// 後で削除する
pi.unregisterProvider("my-llm");
```

登録解除すると、そのプロバイダーの動的モデル、API キーのフォールバック、OAuth プロバイダー登録、カスタムストリームハンドラー登録が削除されます。上書きされていた組み込みモデルやプロバイダー動作も復元されます。

初回の拡張ロード段階を過ぎてからの呼び出しは即時反映されるため、`/reload` は不要です。

### API タイプ

`api` フィールドは、どのストリーミング実装を使うかを決定します。

| API | 用途 |
|-----|------|
| `anthropic-messages` | Anthropic Claude API および互換実装 |
| `openai-completions` | OpenAI Chat Completions API および互換実装 |
| `openai-responses` | OpenAI Responses API |
| `azure-openai-responses` | Azure OpenAI Responses API |
| `openai-codex-responses` | OpenAI Codex Responses API |
| `mistral-conversations` | Mistral SDK の Conversations/Chat ストリーミング |
| `google-generative-ai` | Google Generative AI API |
| `google-vertex` | Google Vertex AI API |
| `bedrock-converse-stream` | Amazon Bedrock Converse API |

ほとんどの OpenAI 互換プロバイダーでは `openai-completions` が使えます。モデルごとに異なる思考レベルが必要なら `thinkingLevelMap` を使い、プロバイダー固有の差異には `compat` を使ってください。

```typescript
models: [{
  id: "custom-model",
  // ...
  reasoning: true,
  thinkingLevelMap: {              // pi のレベルをプロバイダー値に対応付ける。null は未対応レベルを隠す
    minimal: null,
    low: null,
    medium: null,
    high: "default",
    xhigh: "max"
  },
  compat: {
    supportsDeveloperRole: false,   // "developer" ではなく "system" を使う
    supportsReasoningEffort: true,
    maxTokensField: "max_tokens",   // "max_completion_tokens" の代わり
    requiresToolResultName: true,   // ツール結果に name フィールドが必要
    thinkingFormat: "qwen",         // トップレベルに enable_thinking: true
    cacheControlFormat: "anthropic" // Anthropic 形式の cache_control マーカー
  }
}]
```

OpenRouter 形式の `reasoning: { effort }` 制御には `openrouter` を使います。Together 形式の `reasoning: { enabled }` 制御には `together` を使い、`supportsReasoningEffort` も有効なら `reasoning_effort` も送信します。`chat_template_kwargs.enable_thinking` を読むローカル Qwen 互換サーバーには、代わりに `qwen-chat-template` を使ってください。
Anthropic 形式のプロンプトキャッシュを、system プロンプト、最後のツール定義、最後の user/assistant テキスト内容上の `cache_control` で公開している OpenAI 互換プロバイダーには、`cacheControlFormat: "anthropic"` を使ってください。

`api: "anthropic-messages"` を使う Anthropic 互換プロバイダーでは、上流モデルが adaptive thinking（`thinking.type: "adaptive"` と `output_config.effort`）を要求する場合、モデルまたはプロバイダーの `compat.forceAdaptiveThinking: true` を設定してください。組み込みの adaptive Claude モデルではこれは自動設定されます。空の思考署名を返し、再送時に `signature: ""` を期待するプロバイダーに対してのみ、`compat.allowEmptySignature: true` を設定してください。

> 移行メモ: Mistral は `openai-completions` から `mistral-conversations` に移行しました。
> ネイティブの Mistral モデルには `mistral-conversations` を使ってください。
> Mistral 互換またはカスタムエンドポイントを意図的に `openai-completions` 経由にする場合は、必要に応じて `compat` フラグを明示的に設定してください。
