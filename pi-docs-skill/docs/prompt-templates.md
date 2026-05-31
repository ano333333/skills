# プロンプトテンプレート

> 要約: プロンプトテンプレートは、短い Markdown 断片を完全なプロンプトへ展開する仕組みです。このページでは、保存場所、frontmatter 形式、引数ヒント、展開時の引数参照、読み込みルールをまとめています。

プロンプトテンプレートは、完全なプロンプトへ展開される Markdown スニペットです。エディタで `/name` を入力するとテンプレートを呼び出せます。ここで `name` は `.md` を除いたファイル名です。

## 配置場所

Pi は次の場所からプロンプトテンプレートを読み込みます。

- グローバル: `~/.pi/agent/prompts/*.md`
- プロジェクト: `.pi/prompts/*.md`
- パッケージ: `prompts/` ディレクトリ、または `package.json` の `pi.prompts` エントリ
- Settings: ファイルまたはディレクトリを含む `prompts` 配列
- CLI: `--prompt-template <path>`（複数回指定可能）

検出を無効にするには `--no-prompt-templates` を使います。

## 形式

```markdown
---
description: ステージ済みの git 変更をレビュー
---
Review the staged changes (`git diff --cached`). Focus on:
- Bugs and logic errors
- Security issues
- Error handling gaps
```

- ファイル名がコマンド名になります。`review.md` は `/review` になります。
- `description` は任意です。ない場合は、最初の空でない行が使われます。
- `argument-hint` も任意です。設定されている場合、オートコンプリートのドロップダウンで説明の前に表示されます。

### 引数ヒント

frontmatter の `argument-hint` を使うと、オートコンプリートに期待する引数を表示できます。必須引数は `<angle brackets>`、任意引数は `[square brackets]` を使います。

```markdown
---
description: URL から PR をレビューし、構造化された課題とコード分析を行う
argument-hint: "<PR-URL>"
---
```

これはオートコンプリートのドロップダウンで次のように表示されます。

```
→ pr   <PR-URL>       — URL から PR をレビューし、構造化された課題とコード分析を行う
  is   <issue>        — GitHub issue（バグまたは機能要望）を分析する
  wr   [instructions] — 現在のタスクを最初から最後まで完遂する
  cl   — リリース前に changelog エントリを監査する
```

## 使い方

エディタで `/` に続けてテンプレート名を入力します。オートコンプリートには、利用可能なテンプレートと説明が表示されます。

```
/review                           # review.md を展開
/component Button                 # 引数付きで展開
/component Button "click handler" # 複数引数
```

## 引数

テンプレートは位置引数と単純なスライスをサポートします。

- `$1`, `$2`, ... 位置引数
- `$@` または `$ARGUMENTS` は、すべての引数を結合したもの
- `${@:N}` は N 番目以降の引数（1 始まり）
- `${@:N:L}` は N 番目から L 個の引数

例:

```markdown
---
description: コンポーネントを作成する
---
Create a React component named $1 with features: $@
```

使用例: `/component Button "onClick handler" "disabled support"`

## 読み込みルール

- `prompts/` におけるテンプレート検出は再帰的ではありません。
- サブディレクトリ内のテンプレートを使いたい場合は、`prompts` 設定またはパッケージマニフェストで明示的に追加してください。
