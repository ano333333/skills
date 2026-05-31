# カスタムモデル

> 要約: `~/.pi/agent/models.json` を使うと、Ollama や vLLM、LM Studio、各種プロキシ経由のモデルを Pi に追加できます。プロバイダー単位の設定、モデル単位の上書き、互換性フラグや思考レベルの調整まで一つの設定ファイルで管理できます。

`~/.pi/agent/models.json` を通じて、カスタムプロバイダーとモデル（Ollama、vLLM、LM Studio、プロキシなど）を追加できます。

## 目次

- [最小例](#最小例)
- [完全な例](#完全な例)
- [サポートされる API](#サポートされる-api)
- [プロバイダー設定](#プロバイダー設定)
- [モデル設定](#モデル設定)
- [組み込みプロバイダーの上書き](#組み込みプロバイダーの上書き)
- [モデル単位の上書き](#モデル単位の上書き)
- [Anthropic Messages 互換性](#anthropic-messages-互換性)
- [OpenAI 互換性](#openai-互換性)

## 最小例

ローカルモデル（Ollama、LM Studio、vLLM）では、各モデルに必要なのは `id` だけです。

```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://localhost:11434/v1",
      "api": "openai-completions",
      "apiKey": "ollama",
      "models": [
        { "id": "llama3.1:8b" },
        { "id": "qwen2.5-coder:7b" }
      ]
    }
  }
}
```

`apiKey` は必須ですが、Ollama はそれを無視するため、任意の値で動作します。

一部の OpenAI 互換サーバーは、推論対応モデルで使われる `developer` ロールを理解しません。そのようなプロバイダーでは `compat.supportsDeveloperRole` を `false` に設定すると、pi は system プロンプトを `developer` ではなく `system` メッセージとして送信します。サーバーが `reasoning_effort` もサポートしない場合は、`compat.supportsReasoningEffort` も `false` に設定してください。

`compat` はプロバイダーレベルに設定して全モデルへ適用することも、モデルレベルに設定して特定モデルだけ上書きすることもできます。これは Ollama、vLLM、SGLang などの OpenAI 互換サーバーでよく使われます。

```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://localhost:11434/v1",
      "api": "openai-completions",
      "apiKey": "ollama",
      "compat": {
        "supportsDeveloperRole": false,
        "supportsReasoningEffort": false
      },
      "models": [
        {
          "id": "gpt-oss:20b",
          "reasoning": true
        }
      ]
    }
  }
}
```

## 完全な例

特定の値が必要な場合は、デフォルトを上書きしてください。

```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://localhost:11434/v1",
      "api": "openai-completions",
      "apiKey": "ollama",
      "models": [
        {
          "id": "llama3.1:8b",
          "name": "Llama 3.1 8B (Local)",
          "reasoning": false,
          "input": ["text"],
          "contextWindow": 128000,
          "maxTokens": 32000,
          "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 }
        }
      ]
    }
  }
}
```

このファイルは `/model` を開くたびに再読み込みされます。セッション中に編集してもよく、再起動は不要です。

## Google AI Studio の例

`google-generative-ai` と `baseUrl` を使うと、Google AI Studio のモデルや、カスタムの Gemma 4 エントリーを追加できます。

```json
{
  "providers": {
    "my-google": {
      "baseUrl": "https://generativelanguage.googleapis.com/v1beta",
      "api": "google-generative-ai",
      "apiKey": "$GEMINI_API_KEY",
      "models": [
        {
          "id": "gemma-4-31b-it",
          "name": "Gemma 4 31B",
          "input": ["text", "image"],
          "contextWindow": 262144,
          "reasoning": true
        }
      ]
    }
  }
}
```

`google-generative-ai` API タイプへカスタムモデルを追加する場合、`baseUrl` は必須です。

## サポートされる API

| API | 説明 |
|-----|------|
| `openai-completions` | OpenAI Chat Completions（最も互換性が高い） |
| `openai-responses` | OpenAI Responses API |
| `anthropic-messages` | Anthropic Messages API |
| `google-generative-ai` | Google Generative AI |

`api` はプロバイダーレベルに設定して全モデルのデフォルトにすることも、モデルレベルに設定して個別に上書きすることもできます。

## プロバイダー設定

| Field | Description |
|-------|-------------|
| `baseUrl` | API エンドポイント URL |
| `api` | API タイプ（上記参照） |
| `apiKey` | API キー（値の解決方法は後述） |
| `headers` | カスタムヘッダー（値の解決方法は後述） |
| `authHeader` | `true` にすると `Authorization: Bearer <apiKey>` を自動追加 |
| `models` | モデル設定の配列 |
| `modelOverrides` | このプロバイダー上の組み込みモデルに対するモデル単位の上書き |

### 値の解決

`apiKey` と `headers` フィールドは、コマンド実行、環境変数展開、リテラル値をサポートします。

- **シェルコマンド:** 先頭の `"!command"` は値全体をコマンドとして実行し、その標準出力を使います
  ```json
  "apiKey": "!security find-generic-password -ws 'anthropic'"
  "apiKey": "!op read 'op://vault/item/credential'"
  ```
- **環境変数展開:** `"$ENV_VAR"` または `"${ENV_VAR}"` は、その変数の値を使います。より大きなリテラル内での展開も可能です。
  ```json
  "apiKey": "$MY_API_KEY"
  "apiKey": "${KEY_PREFIX}_${KEY_SUFFIX}"
  ```
  `$FOO_BAR` は変数 `FOO_BAR` を意味します。`BAR` をリテラル文字列として扱いたい場合は `${FOO}_BAR` を使ってください。環境変数が未定義なら、その値は未解決になります。
- **エスケープ:** `"$$"` はリテラルの `"$"` を出力し、`"$!"` はコマンド実行を発火させずにリテラルの `"!"` を出力します。
  ```json
  "apiKey": "$$literal-dollar-prefix"
  "apiKey": "$!literal-bang-prefix"
  ```
- **リテラル値:** そのまま使われます
  ```json
  "apiKey": "sk-..."
  ```

`MY_API_KEY` のような従来形式の大文字環境変数風の値は、起動時に `$MY_API_KEY` へ移行されます。

`models.json` では、シェルコマンドはリクエスト時に解決されます。pi は任意コマンドに対して、組み込みの TTL、期限切れ値の再利用、回復ロジックを意図的に適用しません。コマンドごとに適切なキャッシュ戦略や障害時の扱いが異なり、pi にはそれを推測できないためです。

コマンドが遅い、高価、レート制限対象、または一時的な失敗時に以前の値を使い続けたい場合は、必要なキャッシュや TTL 挙動を自前で実装したスクリプトやコマンドで包んでください。

`/model` の利用可否チェックは、設定済みの認証情報の有無だけを見ており、シェルコマンドは実行しません。

### カスタムヘッダー

```json
{
  "providers": {
    "custom-proxy": {
      "baseUrl": "https://proxy.example.com/v1",
      "apiKey": "$MY_API_KEY",
      "api": "anthropic-messages",
      "headers": {
        "x-portkey-api-key": "$PORTKEY_API_KEY",
        "x-secret": "!op read 'op://vault/item/secret'"
      },
      "models": [...]
    }
  }
}
```

## モデル設定

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `id` | Yes | — | モデル識別子（API に渡される） |
| `name` | No | `id` | 人が読むためのモデル名。`--model` パターン照合に使われ、モデル詳細やステータステキストにも表示される。 |
| `api` | No | provider の `api` | このモデルに対してプロバイダーの API を上書きする |
| `reasoning` | No | `false` | 拡張思考をサポートする |
| `thinkingLevelMap` | No | 省略 | pi の思考レベルをプロバイダー値へ対応付け、未対応レベルを示す（後述） |
| `input` | No | `["text"]` | 入力タイプ: `["text"]` または `["text", "image"]` |
| `contextWindow` | No | `128000` | コンテキストウィンドウのトークン数 |
| `maxTokens` | No | `16384` | 最大出力トークン数 |
| `cost` | No | すべて 0 | `{"input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0}`（100 万トークンあたり） |
| `compat` | No | provider の `compat` | プロバイダー互換性の上書き。プロバイダーレベルの `compat` とマージされる。 |

現在の挙動:
- `/model` と `--list-models` はモデル `id` を基準に一覧表示します。
- 設定した `name` はモデル照合と、詳細表示やステータステキストに使われます。

### Thinking Level Map

モデル固有の思考制御を記述するには、モデル上で `thinkingLevelMap` を使います。キーは pi の思考レベル `off`、`minimal`、`low`、`medium`、`high`、`xhigh` です。

値は三値です。

| Value | Meaning |
|-------|---------|
| omitted | そのレベルはサポートされており、プロバイダーのデフォルト対応が使われる |
| string | そのレベルはサポートされており、この値がプロバイダーへ送られる |
| `null` | そのレベルは未対応で、非表示またはスキップまたはクランプ対象になる |

off、高、高度推論のみをサポートするモデルの例です。

```json
{
  "id": "deepseek-v4-pro",
  "reasoning": true,
  "thinkingLevelMap": {
    "minimal": null,
    "low": null,
    "medium": null,
    "high": "high",
    "xhigh": "max"
  }
}
```

思考を無効化できないモデルの例です。

```json
{
  "id": "always-thinking-model",
  "reasoning": true,
  "thinkingLevelMap": {
    "off": null
  }
}
```

移行: 以前の設定で `compat.reasoningEffortMap` を使っていた場合、その対応表はモデルレベルの `thinkingLevelMap` へ移してください。UI に表示したくないレベルには `null` を使ってください。

## 組み込みプロバイダーの上書き

モデルを再定義せずに、組み込みプロバイダーをプロキシ経由に切り替える例です。

```json
{
  "providers": {
    "anthropic": {
      "baseUrl": "https://my-proxy.example.com/v1"
    }
  }
}
```

組み込みの Anthropic モデルはすべて引き続き利用できます。既存の OAuth や API キー認証もそのまま動作します。

カスタムモデルを組み込みプロバイダーへマージしたい場合は、`models` 配列を含めてください。

```json
{
  "providers": {
    "anthropic": {
      "baseUrl": "https://my-proxy.example.com/v1",
      "apiKey": "$ANTHROPIC_API_KEY",
      "api": "anthropic-messages",
      "models": [...]
    }
  }
}
```

マージの意味論:
- 組み込みモデルは保持されます。
- カスタムモデルは、そのプロバイダー内で `id` をキーに upsert されます。
- カスタムモデルの `id` が組み込みモデルの `id` と一致する場合、そのカスタムモデルが組み込みモデルを置き換えます。
- カスタムモデルの `id` が新規なら、組み込みモデルと並んで追加されます。

## モデル単位の上書き

`modelOverrides` を使うと、プロバイダー全体のモデル一覧を置き換えずに、特定の組み込みモデルだけを調整できます。

```json
{
  "providers": {
    "openrouter": {
      "modelOverrides": {
        "anthropic/claude-sonnet-4": {
          "name": "Claude Sonnet 4 (Bedrock Route)",
          "compat": {
            "openRouterRouting": {
              "only": ["amazon-bedrock"]
            }
          }
        }
      }
    }
  }
}
```

`modelOverrides` では、モデルごとに `name`、`reasoning`、`input`、`cost`（部分指定可）、`contextWindow`、`maxTokens`、`headers`、`compat` を指定できます。

挙動メモ:
- `modelOverrides` は組み込みプロバイダーのモデルに適用されます。
- 未知のモデル ID は無視されます。
- プロバイダーレベルの `baseUrl` や `headers` と `modelOverrides` は併用できます。
- プロバイダーに `models` も定義されている場合、カスタムモデルは組み込みモデルへの上書き適用後にマージされます。同じ `id` を持つカスタムモデルは、上書き済みの組み込みモデルエントリーを置き換えます。

## Anthropic Messages 互換性

`api: "anthropic-messages"` を使うプロバイダーやプロキシでは、Anthropic 固有のリクエスト互換性を `compat` で制御できます。

デフォルトでは、pi はツールごとに `eager_input_streaming: true` を送信します。プロキシや Anthropic 互換バックエンドがこのフィールドを拒否する場合は、`supportsEagerToolInputStreaming` を `false` に設定してください。そうすると pi は `tools[].eager_input_streaming` を省略し、代わりにツール有効時のリクエストで旧来の `fine-grained-tool-streaming-2025-05-14` ベータヘッダーを送信します。

一部の Anthropic モデルは、従来の予算ベース思考ペイロードではなく adaptive thinking（`thinking.type: "adaptive"` と `output_config.effort`）を必要とします。組み込みモデルではこれは自動設定されます。そうしたモデルへルーティングするカスタムプロバイダーや別名に対しては、`forceAdaptiveThinking` を `true` に設定してください。

一部の Anthropic 互換プロバイダーは、空の署名を持つ thinking ブロックを出力しつつ、再送時にもそれを期待します。そのようなプロバイダーに対してのみ `allowEmptySignature` を `true` に設定してください。真正の Anthropic は空の thinking 署名を拒否します。

```json
{
  "providers": {
    "anthropic-proxy": {
      "baseUrl": "https://proxy.example.com",
      "api": "anthropic-messages",
      "apiKey": "$ANTHROPIC_PROXY_KEY",
      "compat": {
        "supportsEagerToolInputStreaming": false,
        "supportsLongCacheRetention": true,
        "forceAdaptiveThinking": true,
        "allowEmptySignature": true
      },
      "models": [
        {
          "id": "claude-opus-4-7",
          "reasoning": true,
          "input": ["text", "image"]
        }
      ]
    }
  }
}
```

| Field | Description |
|-------|-------------|
| `supportsEagerToolInputStreaming` | プロバイダーがツールごとの `eager_input_streaming` を受け入れるか。デフォルト: `true`。`false` にするとこのフィールドを省略し、旧来の fine-grained tool streaming ベータヘッダーを使う。 |
| `supportsLongCacheRetention` | キャッシュ保持が `long` のときに、プロバイダーが Anthropic の長期キャッシュ保持（`cache_control.ttl: "1h"`）を受け入れるか。デフォルト: `true`。 |
| `sendSessionAffinityHeaders` | キャッシュ有効時に、セッション ID 由来の `x-session-affinity` を送るか。デフォルトは既知プロバイダーに対する自動判定。 |
| `supportsCacheControlOnTools` | プロバイダーがツール定義上の Anthropic 形式 `cache_control` マーカーを受け入れるか。デフォルト: `true`。 |
| `forceAdaptiveThinking` | このモデルに adaptive thinking（`thinking.type: "adaptive"` と `output_config.effort`）を送るか。組み込みの adaptive モデルでは自動設定。デフォルト: `false`。 |
| `allowEmptySignature` | 空の thinking 署名をテキストへ変換せず、`signature: ""` として再送するか。デフォルト: `false`。 |

## OpenAI 互換性

OpenAI 互換性が部分的なプロバイダーでは、`compat` フィールドを使って差異を埋められます。

- プロバイダーレベルの `compat` は、そのプロバイダー配下の全モデルにデフォルトとして適用されます。
- モデルレベルの `compat` は、そのモデルに対してプロバイダーレベルの値を上書きします。

```json
{
  "providers": {
    "local-llm": {
      "baseUrl": "http://localhost:8080/v1",
      "api": "openai-completions",
      "compat": {
        "supportsUsageInStreaming": false,
        "maxTokensField": "max_tokens"
      },
      "models": [...]
    }
  }
}
```

| Field | Description |
|-------|-------------|
| `supportsStore` | プロバイダーが `store` フィールドをサポートする |
| `supportsDeveloperRole` | `developer` と `system` のどちらのロールを使うか |
| `supportsReasoningEffort` | `reasoning_effort` パラメーターをサポートするか |
| `supportsUsageInStreaming` | `stream_options: { include_usage: true }` をサポートするか（デフォルト: `true`） |
| `maxTokensField` | `max_completion_tokens` または `max_tokens` のどちらを使うか |
| `requiresToolResultName` | ツール結果メッセージに `name` を含めるか |
| `requiresAssistantAfterToolResult` | ツール結果の後に user メッセージを送る前に assistant メッセージを挿入するか |
| `requiresThinkingAsText` | thinking ブロックをプレーンテキストへ変換するか |
| `requiresReasoningContentOnAssistantMessages` | 推論有効時に、再送されるすべての assistant メッセージへ空の `reasoning_content` を含めるか |
| `thinkingFormat` | `reasoning_effort`、`openrouter`、`deepseek`、`together`、`zai`、`qwen`、`qwen-chat-template` のどの思考パラメーター形式を使うか |
| `cacheControlFormat` | system プロンプト、最後のツール定義、最後の user/assistant テキスト内容に Anthropic 形式の `cache_control` マーカーを使うか。現状では `anthropic` のみサポート。 |
| `supportsStrictMode` | ツール定義に `strict` フィールドを含めるか |
| `supportsLongCacheRetention` | キャッシュ保持が `long` のときに、長期キャッシュ保持を受け入れるか。OpenAI プロンプトキャッシュでは `prompt_cache_retention: "24h"`、`cacheControlFormat` が `anthropic` の場合は `cache_control.ttl: "1h"` を使う。デフォルト: `true`。 |
| `openRouterRouting` | OpenRouter のプロバイダールーティング設定。このオブジェクトは [OpenRouter API リクエスト](https://openrouter.ai/docs/guides/routing/provider-selection) の `provider` フィールドへそのまま送られる。 |
| `vercelGatewayRouting` | Vercel AI Gateway のプロバイダー選択ルーティング設定（`only`、`order`） |

`openrouter` は `reasoning: { effort }` を使います。`together` は `reasoning: { enabled }` を使い、`supportsReasoningEffort` が有効なら `reasoning_effort` も送ります。`qwen` はトップレベルの `enable_thinking` を使います。`chat_template_kwargs.enable_thinking` が必要なローカル Qwen 互換サーバーには `qwen-chat-template` を使ってください。

`cacheControlFormat: "anthropic"` は、テキスト内容やツール定義上の `cache_control` マーカーを通じて Anthropic 形式のプロンプトキャッシュを公開する OpenAI 互換プロバイダー向けです。

例:

```json
{
  "providers": {
    "openrouter": {
      "baseUrl": "https://openrouter.ai/api/v1",
      "apiKey": "$OPENROUTER_API_KEY",
      "api": "openai-completions",
      "models": [
        {
          "id": "openrouter/anthropic/claude-3.5-sonnet",
          "name": "OpenRouter Claude 3.5 Sonnet",
          "compat": {
            "openRouterRouting": {
              "allow_fallbacks": true,
              "require_parameters": false,
              "data_collection": "deny",
              "zdr": true,
              "enforce_distillable_text": false,
              "order": ["anthropic", "amazon-bedrock", "google-vertex"],
              "only": ["anthropic", "amazon-bedrock"],
              "ignore": ["gmicloud", "friendli"],
              "quantizations": ["fp16", "bf16"],
              "sort": {
                "by": "price",
                "partition": "model"
              },
              "max_price": {
                "prompt": 10,
                "completion": 20
              },
              "preferred_min_throughput": {
                "p50": 100,
                "p90": 50
              },
              "preferred_max_latency": {
                "p50": 1,
                "p90": 3,
                "p99": 5
              }
            }
          }
        }
      ]
    }
  }
}
```

Vercel AI Gateway の例:

```json
{
  "providers": {
    "vercel-ai-gateway": {
      "baseUrl": "https://ai-gateway.vercel.sh/v1",
      "apiKey": "$AI_GATEWAY_API_KEY",
      "api": "openai-completions",
      "models": [
        {
          "id": "moonshotai/kimi-k2.5",
          "name": "Kimi K2.5 (Fireworks via Vercel)",
          "reasoning": true,
          "input": ["text", "image"],
          "cost": { "input": 0.6, "output": 3, "cacheRead": 0, "cacheWrite": 0 },
          "contextWindow": 262144,
          "maxTokens": 262144,
          "compat": {
            "vercelGatewayRouting": {
              "only": ["fireworks", "novita"],
              "order": ["fireworks", "novita"]
            }
          }
        }
      ]
    }
  }
}
```
