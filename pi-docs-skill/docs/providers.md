# プロバイダー

> 要約: Pi は、OAuth ベースのサブスクリプション型プロバイダーと、API キー型プロバイダーの両方をサポートします。認証情報の保存先、クラウドプロバイダーごとの設定方法、カスタムプロバイダー導入時の分岐まで、このページで全体像を確認できます。

Pi は、OAuth を使うサブスクリプション型プロバイダーと、環境変数または認証ファイルを使う API キー型プロバイダーをサポートします。各プロバイダーについて、pi は利用可能なすべてのモデルを把握しています。この一覧は pi のリリースごとに更新されます。

## 目次

- [サブスクリプション](#サブスクリプション)
- [API キー](#api-キー)
- [認証ファイル](#認証ファイル)
- [クラウドプロバイダー](#クラウドプロバイダー)
- [カスタムプロバイダー](#カスタムプロバイダー)
- [解決順序](#解決順序)

## サブスクリプション

対話モードで `/login` を使い、プロバイダーを選択します。

- ChatGPT Plus/Pro (Codex)
- Claude Pro/Max
- GitHub Copilot

認証情報を消去するには `/logout` を使います。トークンは `~/.pi/agent/auth.json` に保存され、期限切れ時には自動更新されます。

### OpenAI Codex

- ChatGPT Plus または Pro の契約が必要です
- OpenAI が公式に後押ししています: [Codex for OSS](https://developers.openai.com/community/codex-for-oss)

### Claude Pro/Max

Anthropic サブスクリプション認証は、Claude Pro/Max アカウントで有効です。サードパーティ製ハーネスからの利用は [extra usage](https://claude.ai/settings/usage) を消費し、Claude プラン上限ではなくトークンごとの課金になります。

### GitHub Copilot

- `github.com` を使う場合は Enter を押すか、GitHub Enterprise Server のドメインを入力します
- "model not supported" と表示されたら、VS Code で有効化してください: Copilot Chat → モデルセレクター → モデルを選択 → "Enable"

## API キー

### 環境変数または認証ファイル

対話モードで `/login` を使ってプロバイダーを選ぶと、API キーを `auth.json` に保存できます。あるいは環境変数で認証情報を設定することもできます。

```bash
export ANTHROPIC_API_KEY=sk-ant-...
pi
```

| Provider | Environment Variable | `auth.json` key |
|----------|----------------------|------------------|
| Anthropic | `ANTHROPIC_API_KEY` | `anthropic` |
| Azure OpenAI Responses | `AZURE_OPENAI_API_KEY` | `azure-openai-responses` |
| OpenAI | `OPENAI_API_KEY` | `openai` |
| DeepSeek | `DEEPSEEK_API_KEY` | `deepseek` |
| Google Gemini | `GEMINI_API_KEY` | `google` |
| Mistral | `MISTRAL_API_KEY` | `mistral` |
| Groq | `GROQ_API_KEY` | `groq` |
| Cerebras | `CEREBRAS_API_KEY` | `cerebras` |
| Cloudflare AI Gateway | `CLOUDFLARE_API_KEY` (+ `CLOUDFLARE_ACCOUNT_ID`, `CLOUDFLARE_GATEWAY_ID`) | `cloudflare-ai-gateway` |
| Cloudflare Workers AI | `CLOUDFLARE_API_KEY` (+ `CLOUDFLARE_ACCOUNT_ID`) | `cloudflare-workers-ai` |
| xAI | `XAI_API_KEY` | `xai` |
| OpenRouter | `OPENROUTER_API_KEY` | `openrouter` |
| Vercel AI Gateway | `AI_GATEWAY_API_KEY` | `vercel-ai-gateway` |
| ZAI | `ZAI_API_KEY` | `zai` |
| OpenCode Zen | `OPENCODE_API_KEY` | `opencode` |
| OpenCode Go | `OPENCODE_API_KEY` | `opencode-go` |
| Hugging Face | `HF_TOKEN` | `huggingface` |
| Fireworks | `FIREWORKS_API_KEY` | `fireworks` |
| Together AI | `TOGETHER_API_KEY` | `together` |
| Kimi For Coding | `KIMI_API_KEY` | `kimi-coding` |
| MiniMax | `MINIMAX_API_KEY` | `minimax` |
| MiniMax (China) | `MINIMAX_CN_API_KEY` | `minimax-cn` |
| Xiaomi MiMo | `XIAOMI_API_KEY` | `xiaomi` |
| Xiaomi MiMo Token Plan (China) | `XIAOMI_TOKEN_PLAN_CN_API_KEY` | `xiaomi-token-plan-cn` |
| Xiaomi MiMo Token Plan (Amsterdam) | `XIAOMI_TOKEN_PLAN_AMS_API_KEY` | `xiaomi-token-plan-ams` |
| Xiaomi MiMo Token Plan (Singapore) | `XIAOMI_TOKEN_PLAN_SGP_API_KEY` | `xiaomi-token-plan-sgp` |

環境変数と `auth.json` キーの対応は、[`packages/ai/src/env-api-keys.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/ai/src/env-api-keys.ts) の [`const envMap`](https://github.com/earendil-works/pi-mono/blob/main/packages/ai/src/env-api-keys.ts) を参照してください。

#### 認証ファイル

認証情報は `~/.pi/agent/auth.json` に保存できます。

```json
{
  "anthropic": { "type": "api_key", "key": "sk-ant-..." },
  "openai": { "type": "api_key", "key": "sk-..." },
  "deepseek": { "type": "api_key", "key": "sk-..." },
  "google": { "type": "api_key", "key": "..." },
  "opencode": { "type": "api_key", "key": "..." },
  "opencode-go": { "type": "api_key", "key": "..." },
  "together": { "type": "api_key", "key": "..." },
  "xiaomi": { "type": "api_key", "key": "..." },
  "xiaomi-token-plan-cn":  { "type": "api_key", "key": "..." },
  "xiaomi-token-plan-ams": { "type": "api_key", "key": "..." },
  "xiaomi-token-plan-sgp": { "type": "api_key", "key": "..." }
}
```

このファイルは `0600` パーミッション（ユーザーのみ読み書き可）で作成されます。認証ファイル内の認証情報は、環境変数より優先されます。

### キーの解決

`key` フィールドは、コマンド実行、環境変数展開、リテラル値をサポートします。

- **シェルコマンド:** 先頭の `"!command"` は値全体をコマンドとして実行し、その標準出力を使います（プロセス生存期間中キャッシュされます）
  ```json
  { "type": "api_key", "key": "!security find-generic-password -ws 'anthropic'" }
  { "type": "api_key", "key": "!op read 'op://vault/item/credential'" }
  ```
- **環境変数展開:** `"$ENV_VAR"` または `"${ENV_VAR}"` は、その名前の変数値を使います。より大きなリテラル内での展開も可能です。
  ```json
  { "type": "api_key", "key": "$MY_ANTHROPIC_KEY" }
  { "type": "api_key", "key": "${KEY_PREFIX}_${KEY_SUFFIX}" }
  ```
  `$FOO_BAR` は変数 `FOO_BAR` を意味します。`BAR` をリテラル文字列にしたい場合は `${FOO}_BAR` を使ってください。環境変数が未定義なら、その値は未解決になります。
- **エスケープ:** `"$$"` はリテラルの `"$"` を出力し、`"$!"` はコマンド実行を発火させずにリテラルの `"!"` を出力します。
  ```json
  { "type": "api_key", "key": "$$literal-dollar-prefix" }
  { "type": "api_key", "key": "$!literal-bang-prefix" }
  ```
- **リテラル値:** そのまま使われます
  ```json
  { "type": "api_key", "key": "sk-ant-..." }
  { "type": "api_key", "key": "public" }
  ```

`MY_API_KEY` のような従来形式の大文字環境変数風の値は、起動時に `$MY_API_KEY` へ移行されます。OAuth 認証情報も `/login` 後にここへ保存され、自動管理されます。

## クラウドプロバイダー

### Azure OpenAI

```bash
export AZURE_OPENAI_API_KEY=...
export AZURE_OPENAI_BASE_URL=https://your-resource.openai.azure.com
# こちらもサポート: https://your-resource.cognitiveservices.azure.com
# ルートエンドポイントは自動で /openai/v1 に正規化される
# あるいは base URL の代わりにリソース名を使ってもよい
export AZURE_OPENAI_RESOURCE_NAME=your-resource

# Optional
export AZURE_OPENAI_API_VERSION=2024-02-01
export AZURE_OPENAI_DEPLOYMENT_NAME_MAP=gpt-4=my-gpt4,gpt-4o=my-gpt4o
```

### Amazon Bedrock

```bash
# Option 1: AWS Profile
export AWS_PROFILE=your-profile

# Option 2: IAM Keys
export AWS_ACCESS_KEY_ID=AKIA...
export AWS_SECRET_ACCESS_KEY=...

# Option 3: Bearer Token
export AWS_BEARER_TOKEN_BEDROCK=...

# Optional region (defaults to us-east-1)
export AWS_REGION=us-west-2
```

ECS タスクロール（`AWS_CONTAINER_CREDENTIALS_*`）と IRSA（`AWS_WEB_IDENTITY_TOKEN_FILE`）もサポートしています。

```bash
pi --provider amazon-bedrock --model us.anthropic.claude-sonnet-4-20250514-v1:0
```

Claude モデルで、ID に認識可能なモデル名が含まれている場合（ベースモデルおよび system-defined inference profile）、プロンプトキャッシュは自動で有効になります。アプリケーション inference profile のように ARN にモデル名が含まれない場合は、`AWS_BEDROCK_FORCE_CACHE=1` を設定してキャッシュポイントを有効化してください。

```bash
export AWS_BEDROCK_FORCE_CACHE=1
pi --provider amazon-bedrock --model arn:aws:bedrock:us-east-1:123456789012:application-inference-profile/abc123
```

Bedrock API プロキシへ接続する場合は、以下の環境変数を使えます。

```bash
# Bedrock プロキシの URL を設定する（標準 AWS SDK 環境変数）
export AWS_ENDPOINT_URL_BEDROCK_RUNTIME=https://my.corp.proxy/bedrock

# プロキシが認証を必要としない場合に設定する
export AWS_BEDROCK_SKIP_AUTH=1

# プロキシが HTTP/1.1 のみをサポートする場合に設定する
export AWS_BEDROCK_FORCE_HTTP1=1
```

### Cloudflare AI Gateway

`CLOUDFLARE_API_KEY` は `/login` で設定できます。アカウント ID と gateway slug は環境変数で設定する必要があります。

```bash
export CLOUDFLARE_API_KEY=...           # or use /login
export CLOUDFLARE_ACCOUNT_ID=...
export CLOUDFLARE_GATEWAY_ID=...        # create at dash.cloudflare.com → AI → AI Gateway
pi --provider cloudflare-ai-gateway --model "claude-sonnet-4-5"
```

OpenAI、Anthropic、Workers AI へのリクエストを Cloudflare AI Gateway 経由でルーティングします。Workers AI は Unified API（`/compat`）と、`workers-ai/@cf/...` という接頭辞付きモデル ID を使います。OpenAI は OpenAI パススルールート（`/openai`）と `gpt-5.1` のようなネイティブ OpenAI モデル ID を使います。Anthropic は Anthropic パススルールート（`/anthropic`）と `claude-sonnet-4-5` のようなネイティブ Anthropic モデル ID を使います。

AI Gateway の認証では `CLOUDFLARE_API_KEY` を `cf-aig-authorization` として使います。上流認証は次のいずれかです。

| Mode | Request auth | Upstream auth |
|------|--------------|---------------|
| Workers AI | Cloudflare token only | Cloudflare-native |
| Unified billing | Cloudflare token only | Cloudflare が上流認証を処理し、クレジットを差し引く |
| Stored BYOK | Cloudflare token only | Cloudflare が AI Gateway ダッシュボードに保存されたプロバイダーキーを注入する |
| Inline BYOK | Cloudflare token + upstream `Authorization` header | リクエスト自身が上流プロバイダーキーを供給する |

通常の pi 利用では、Unified billing または Stored BYOK を推奨します。Inline BYOK では、Cloudflare AI Gateway プロバイダーに対して追加の上流 `Authorization` ヘッダー設定が必要です。たとえば `models.json` のプロバイダーまたはモデル上書きを使います。

### Cloudflare Workers AI

`CLOUDFLARE_API_KEY` は `/login` で設定できます。`CLOUDFLARE_ACCOUNT_ID` は環境変数で設定する必要があります。

```bash
export CLOUDFLARE_API_KEY=...           # or use /login
export CLOUDFLARE_ACCOUNT_ID=...
pi --provider cloudflare-workers-ai --model "@cf/moonshotai/kimi-k2.6"
```

Pi は、[prefix caching](https://developers.cloudflare.com/workers-ai/features/prompt-caching/) の割引を受けるために `x-session-affinity` を自動設定します。

### Google Vertex AI

Application Default Credentials を使います。

```bash
gcloud auth application-default login
export GOOGLE_CLOUD_PROJECT=your-project
export GOOGLE_CLOUD_LOCATION=us-central1
```

または、`GOOGLE_APPLICATION_CREDENTIALS` にサービスアカウントキーファイルを設定してください。

## カスタムプロバイダー

**models.json 経由:** Ollama、LM Studio、vLLM、またはサポート対象 API（OpenAI Completions、OpenAI Responses、Anthropic Messages、Google Generative AI）を話せる任意のプロバイダーを追加できます。詳細は [models.md](models.md) を参照してください。

**拡張機能経由:** カスタム API 実装や OAuth フローが必要なプロバイダーでは、拡張機能を作成してください。詳細は [custom-provider.md](custom-provider.md) と [examples/extensions/custom-provider-gitlab-duo](../examples/extensions/custom-provider-gitlab-duo/) を参照してください。

## 解決順序

あるプロバイダーの認証情報を解決するとき、優先順位は次のとおりです。

1. CLI の `--api-key` フラグ
2. `auth.json` エントリー（API キーまたは OAuth トークン）
3. 環境変数
4. `models.json` のカスタムプロバイダーキー
