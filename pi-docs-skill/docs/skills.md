# スキル

> 要約: スキルは、エージェントが必要時に読み込む自己完結型の機能パッケージです。このページでは、スキルの配置場所、読み込みの仕組み、`SKILL.md` の構造、frontmatter の要件、検証ルール、共有リポジトリの扱いを説明しています。

スキルは、エージェントがオンデマンドで読み込む自己完結型の機能パッケージです。スキルは、特定タスク向けの専門ワークフロー、セットアップ手順、補助スクリプト、参照ドキュメントを提供します。

Pi は [Agent Skills standard](https://agentskills.io/specification) を実装しており、ほとんどの違反には警告を出しつつも、厳格すぎない挙動を取ります。Pi では、親ディレクトリ名と異なるスキル名を許可しています。標準では禁止されていますが、そのルールは複数のエージェントハーネス間で共有されるスキルディレクトリには適していません。

## 目次

- [配置場所](#配置場所)
- [スキルの動作](#スキルの動作)
- [スキルコマンド](#スキルコマンド)
- [スキル構造](#スキル構造)
- [Frontmatter](#frontmatter)
- [バリデーション](#バリデーション)
- [例](#例)
- [スキルリポジトリ](#スキルリポジトリ)

## 配置場所

> **セキュリティ:** スキルはモデルに任意の操作を指示でき、モデルが呼び出す実行可能コードを含む場合もあります。使用前にスキル内容を確認してください。

Pi は次の場所からスキルを読み込みます。

- グローバル:
  - `~/.pi/agent/skills/`
  - `~/.agents/skills/`
- プロジェクト:
  - `.pi/skills/`
  - `cwd` および祖先ディレクトリの `.agents/skills/`（git リポジトリ内ならリポジトリルートまで、そうでなければファイルシステムルートまで）
- パッケージ: `skills/` ディレクトリ、または `package.json` の `pi.skills` エントリ
- Settings: ファイルまたはディレクトリを含む `skills` 配列
- CLI: `--skill <path>`（複数回指定可能。`--no-skills` と併用しても追加で読み込まれます）

検出ルール:
- `~/.pi/agent/skills/` と `.pi/skills/` では、ルート直下の `.md` ファイルが個別スキルとして検出されます
- すべてのスキル配置場所で、`SKILL.md` を含むディレクトリは再帰的に検出されます
- `~/.agents/skills/` とプロジェクトの `.agents/skills/` では、ルート直下の `.md` ファイルは無視されます

検出を無効にするには `--no-skills` を使います（明示的な `--skill` パスは引き続き読み込まれます）。

### 他のハーネスのスキルを使う

Claude Code や OpenAI Codex のスキルを使うには、それらのディレクトリを設定に追加します。

```json
{
  "skills": [
    "~/.claude/skills",
    "~/.codex/skills"
  ]
}
```

プロジェクトレベルの Claude Code スキルを使う場合は、`.pi/settings.json` に追加します。

```json
{
  "skills": ["../.claude/skills"]
}
```

## スキルの動作

1. 起動時に、pi はスキル配置場所を走査して名前と説明を抽出します
2. システムプロンプトには、[specification](https://agentskills.io/integrate-skills) に従った XML 形式で利用可能スキルが含まれます
3. タスクが一致すると、エージェントは `read` を使って完全な `SKILL.md` を読み込みます（モデルが必ずそうするとは限りません。確実に読み込ませたい場合はプロンプトや `/skill:name` を使ってください）
4. エージェントは指示に従い、スクリプトやアセットを参照する際は相対パスを使います

これは progressive disclosure です。説明だけは常にコンテキストに入り、完全な手順は必要時にだけ読み込まれます。

## スキルコマンド

スキルは `/skill:name` コマンドとして登録されます。

```bash
/skill:brave-search           # スキルを読み込んで実行
/skill:pdf-tools extract      # 引数付きでスキルを読み込む
```

コマンドの後ろの引数は、`User: <args>` としてスキル内容に追記されます。

対話モードでは `/settings`、または `settings.json` でスキルコマンドの有効化を切り替えられます。

```json
{
  "enableSkillCommands": true
}
```

## スキル構造

スキルは `SKILL.md` ファイルを持つディレクトリです。それ以外の構成は自由です。

```
my-skill/
├── SKILL.md              # 必須: frontmatter + 手順
├── scripts/              # 補助スクリプト
│   └── process.sh
├── references/           # 必要時に読み込む詳細ドキュメント
│   └── api-reference.md
└── assets/
    └── template.json
```

### SKILL.md 形式

````markdown
---
name: my-skill
description: このスキルが何をし、いつ使うべきか。具体的に書く。
---

# My Skill

## Setup

初回利用前に一度だけ実行:
```bash
cd /path/to/skill && npm install
```

## Usage

```bash
./scripts/process.sh <input>
```
````

スキルディレクトリからの相対パスを使います。

```markdown
詳しくは [reference guide](references/REFERENCE.md) を参照してください。
```

## Frontmatter

[Agent Skills specification](https://agentskills.io/specification#frontmatter-required) に従うと:

| Field | Required | 説明 |
|-------|----------|------|
| `name` | Yes | 最大 64 文字。小文字 a-z、0-9、ハイフン。標準とは異なり、Pi は親ディレクトリ名との一致を要求しません。これは共有スキルディレクトリでは標準の要件が適切でないためです。 |
| `description` | Yes | 最大 1024 文字。このスキルが何をし、いつ使うか。 |
| `license` | No | ライセンス名、または同梱ファイルへの参照。 |
| `compatibility` | No | 最大 500 文字。環境要件。 |
| `metadata` | No | 任意のキーと値のマッピング。 |
| `allowed-tools` | No | 事前承認済みツールの空白区切りリスト（実験的）。 |
| `disable-model-invocation` | No | `true` の場合、システムプロンプトから隠されます。ユーザーは `/skill:name` を使う必要があります。 |

### 名前のルール

- 1〜64 文字
- 小文字、数字、ハイフンのみ
- 先頭・末尾にハイフン不可
- ハイフンの連続不可
Pi は、名前が親ディレクトリと一致することを要求しません。Agent Skills 標準では要求していますが、その要件は複数ツールで共有されるスキルディレクトリには適していません。

有効: `pdf-processing`, `data-analysis`, `code-review`
無効: `PDF-Processing`, `-pdf`, `pdf--processing`

### 説明文のベストプラクティス

説明文によって、エージェントがいつスキルを読み込むかが決まります。具体的に書いてください。

良い例:
```yaml
description: PDF ファイルからテキストと表を抽出し、PDF フォームを入力し、複数の PDF を結合します。PDF ドキュメントを扱うときに使用します。
```

悪い例:
```yaml
description: PDF を手伝います。
```

## バリデーション

Pi は Agent Skills standard に照らしてスキルを検証します。ほとんどの問題は警告になりますが、スキル自体は読み込まれます。

- 名前が 64 文字を超える、または無効な文字を含む
- 名前の先頭または末尾がハイフン、または連続ハイフンを含む
- 説明が 1024 文字を超える

未知の frontmatter フィールドは無視されます。

**例外:** 説明が欠けているスキルは読み込まれません。

名前衝突（異なる場所から同名スキルが見つかる）では警告を出し、最初に見つかったスキルが維持されます。

## 例

```
brave-search/
├── SKILL.md
├── search.js
└── content.js
```

**SKILL.md:**
````markdown
---
name: brave-search
description: Brave Search API を使った Web 検索とコンテンツ抽出。ドキュメント、事実確認、その他の Web コンテンツ検索に使用します。
---

# Brave Search

## Setup

```bash
cd /path/to/brave-search && npm install
```

## Search

```bash
./search.js "query"              # 基本検索
./search.js "query" --content    # ページ内容も含める
```

## Extract Page Content

```bash
./content.js https://example.com
```
````

## スキルリポジトリ

- [Anthropic Skills](https://github.com/anthropics/skills) - ドキュメント処理（docx, pdf, pptx, xlsx）、Web 開発
- [Pi Skills](https://github.com/badlogic/pi-skills) - Web 検索、ブラウザ自動化、Google APIs、文字起こし
