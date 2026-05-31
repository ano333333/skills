# Pi パッケージ

> 要約: Pi パッケージは、拡張機能、スキル、プロンプトテンプレート、テーマをひとまとまりにして npm や git 経由で共有する仕組みです。インストール元、パッケージ構造、依存関係、読み込みフィルタ、スコープの競合解決までをこのページで定義しています。

Pi パッケージは、拡張機能、スキル、プロンプトテンプレート、テーマを束ね、npm または git を通じて共有できるようにします。パッケージは `package.json` の `pi` キー配下でリソースを宣言することも、慣例ディレクトリを使うこともできます。

## 目次

- [インストールと管理](#インストールと管理)
- [パッケージソース](#パッケージソース)
- [Pi パッケージの作成](#pi-パッケージの作成)
- [パッケージ構造](#パッケージ構造)
- [依存関係](#依存関係)
- [パッケージフィルタリング](#パッケージフィルタリング)
- [リソースの有効化と無効化](#リソースの有効化と無効化)
- [スコープと重複排除](#スコープと重複排除)

## インストールと管理

> **セキュリティ:** Pi パッケージはシステムへの完全なアクセス権で実行されます。拡張機能は任意のコードを実行でき、スキルは実行ファイルの起動を含む任意の操作をモデルに指示できます。サードパーティ製パッケージをインストールする前に、必ずソースコードを確認してください。

```bash
pi install npm:@foo/bar@1.0.0
pi install git:github.com/user/repo@v1
pi install https://github.com/user/repo  # 生の URL でも動作します
pi install /absolute/path/to/package
pi install ./relative/path/to/package

pi remove npm:@foo/bar
pi list                     # settings に登録されたインストール済みパッケージを表示
pi update                   # pi 本体、パッケージ、固定済み git ref の整合性を更新
pi update --extensions      # パッケージと固定済み git ref の整合性のみ更新
pi update --self            # pi 本体のみ更新
pi update --self --force    # 現在版でも pi を再インストール
pi update npm:@foo/bar      # 単一パッケージを更新
pi update --extension npm:@foo/bar
```

これらのコマンドが管理するのは pi CLI 自体ではなく、Pi パッケージです。pi 本体のアンインストールについては [Quickstart](quickstart.md#uninstall) を参照してください。

デフォルトでは、`install` と `remove` はユーザー設定（`~/.pi/agent/settings.json`）に書き込みます。代わりに `-l` を使うとプロジェクト設定（`.pi/settings.json`）に書き込みます。プロジェクト設定はチームと共有でき、pi は起動時に不足しているパッケージを自動的にインストールします。

パッケージをインストールせずに試したい場合は、`--extension` または `-e` を使います。これにより、その実行に限って一時ディレクトリへインストールされます。

```bash
pi -e npm:@foo/bar
pi -e git:github.com/user/repo
```

## パッケージソース

Pi は、設定および `pi install` において 3 種類のソースを受け付けます。

### npm

```
npm:@scope/pkg@1.2.3
npm:pkg
```

- バージョン付き指定は固定され、パッケージ更新（`pi update`, `pi update --extensions`）ではスキップされます。
- ユーザー単位のインストール先は `~/.pi/agent/npm/` です。
- プロジェクト単位のインストール先は `.pi/npm/` です。
- npm パッケージの検索とインストールを `mise` や `asdf` のような特定ラッパーコマンドに固定したい場合は、`settings.json` の `npmCommand` を設定します。

例:

```json
{
  "npmCommand": ["mise", "exec", "node@20", "--", "npm"]
}
```

### git

```
git:github.com/user/repo@v1
git:git@github.com:user/repo@v1
https://github.com/user/repo@v1
ssh://git@github.com/user/repo@v1
```

- `git:` プレフィックスなしでは、プロトコル付き URL（`https://`, `http://`, `ssh://`, `git://`）だけが受け付けられます。
- `git:` プレフィックス付きでは、`github.com/user/repo` や `git@github.com:user/repo` を含む短縮形式も使えます。
- HTTPS と SSH の両方の URL をサポートします。
- SSH URL では、設定済みの SSH キーが自動的に使われます（`~/.ssh/config` を尊重します）。
- 非対話実行（たとえば CI）では、`GIT_TERMINAL_PROMPT=0` を設定して認証情報プロンプトを無効化し、`GIT_SSH_COMMAND`（たとえば `ssh -o BatchMode=yes -o ConnectTimeout=5`）を設定して即座に失敗させることができます。
- ref は固定されたタグまたはコミットです。`pi update` と `pi update --extensions` はそれを新しい ref に進めませんが、既存 clone を設定済み ref に整合させます。
- `pi install git:host/user/repo@new-ref` を使うと、設定を更新し、既存パッケージを新しい固定 ref へ移動できます。
- clone 先は `~/.pi/agent/git/<host>/<path>`（グローバル）または `.pi/git/<host>/<path>`（プロジェクト）です。
- 整合処理でチェックアウト内容が変わった場合、pi は clone を reset/clean したうえで、`package.json` が存在すれば `npm install` を実行します。

**SSH の例:**
```bash
# git@host:path の短縮形（git: プレフィックスが必要）
pi install git:git@github.com:user/repo

# ssh:// プロトコル形式
pi install ssh://git@github.com/user/repo

# バージョン ref 付き
pi install git:git@github.com:user/repo@v1.0.0
```

### ローカルパス

```
/absolute/path/to/package
./relative/path/to/package
```

ローカルパスはディスク上のファイルまたはディレクトリを指し、コピーせずに設定へ追加されます。相対パスは、それが記述された設定ファイルを基準に解決されます。パスがファイルなら単一の拡張機能として読み込まれ、ディレクトリならパッケージルールに従ってリソースが読み込まれます。

## Pi パッケージの作成

`package.json` に `pi` マニフェストを追加するか、慣例ディレクトリを使用します。見つけやすくするために `pi-package` キーワードを含めてください。

```json
{
  "name": "my-package",
  "keywords": ["pi-package"],
  "pi": {
    "extensions": ["./extensions"],
    "skills": ["./skills"],
    "prompts": ["./prompts"],
    "themes": ["./themes"]
  }
}
```

パスはパッケージルートからの相対です。配列では glob パターンと `!除外` を使えます。

### ギャラリーメタデータ

[package gallery](https://pi.dev/packages) には、`pi-package` タグを持つパッケージが表示されます。プレビューを表示するには `video` または `image` フィールドを追加します。

```json
{
  "name": "my-package",
  "keywords": ["pi-package"],
  "pi": {
    "extensions": ["./extensions"],
    "video": "https://example.com/demo.mp4",
    "image": "https://example.com/screenshot.png"
  }
}
```

- **video**: MP4 のみ。デスクトップではホバー時に自動再生され、クリックで全画面プレーヤーが開きます。
- **image**: PNG、JPEG、GIF、WebP を使用できます。静的プレビューとして表示されます。

両方が設定されている場合は、video が優先されます。

## パッケージ構造

### 慣例ディレクトリ

`pi` マニフェストが存在しない場合、pi は以下のディレクトリからリソースを自動検出します。

- `extensions/` は `.ts` と `.js` ファイルを読み込みます
- `skills/` は `SKILL.md` を持つフォルダを再帰的に見つけ、トップレベルの `.md` ファイルもスキルとして読み込みます
- `prompts/` は `.md` ファイルを読み込みます
- `themes/` は `.json` ファイルを読み込みます

## 依存関係

サードパーティの実行時依存関係は `package.json` の `dependencies` に置きます。拡張機能、スキル、プロンプトテンプレート、テーマを登録しない依存関係も `dependencies` に含めます。pi が npm または git からパッケージをインストールするとき、`npm install` を実行するため、それらの依存関係も自動的にインストールされます。

Pi は、拡張機能とスキル向けのコアパッケージをバンドルしています。以下のいずれかを import する場合は、`peerDependencies` に `"*"` レンジで記載し、バンドルしないでください: `@earendil-works/pi-ai`, `@earendil-works/pi-agent-core`, `@earendil-works/pi-coding-agent`, `@earendil-works/pi-tui`, `typebox`。

その他の Pi パッケージは tarball に同梱する必要があります。`dependencies` と `bundledDependencies` に追加し、`node_modules/` 配下のパス経由でそれらのリソースを参照してください。Pi はパッケージごとに分離されたモジュールルートで読み込むため、別々のインストール同士で衝突したり、モジュールを共有したりしません。

例:

```json
{
  "dependencies": {
    "shitty-extensions": "^1.0.1"
  },
  "bundledDependencies": ["shitty-extensions"],
  "pi": {
    "extensions": ["extensions", "node_modules/shitty-extensions/extensions"],
    "skills": ["skills", "node_modules/shitty-extensions/skills"]
  }
}
```

## パッケージフィルタリング

設定でオブジェクト形式を使うと、パッケージが何を読み込むかを絞り込めます。

```json
{
  "packages": [
    "npm:simple-pkg",
    {
      "source": "npm:my-package",
      "extensions": ["extensions/*.ts", "!extensions/legacy.ts"],
      "skills": [],
      "prompts": ["prompts/review.md"],
      "themes": ["+themes/legacy.json"]
    }
  ]
}
```

`+path` と `-path` は、パッケージルートからの正確な相対パスです。

- キーを省略すると、その種類はすべて読み込まれます。
- `[]` を使うと、その種類は何も読み込みません。
- `!pattern` は一致項目を除外します。
- `+path` は正確なパスを強制的に含めます。
- `-path` は正確なパスを強制的に除外します。
- フィルタはマニフェストの上に重ねて適用されます。すでに許可されている範囲をさらに狭めるものです。

## リソースの有効化と無効化

`pi config` を使うと、インストール済みパッケージやローカルディレクトリ由来の拡張機能、スキル、プロンプトテンプレート、テーマを有効化または無効化できます。グローバルスコープ（`~/.pi/agent`）とプロジェクトスコープ（`.pi/`）の両方で動作します。

## スコープと重複排除

パッケージはグローバル設定とプロジェクト設定の両方に現れることがあります。同じパッケージが両方に存在する場合は、プロジェクト側のエントリが優先されます。識別方法は次のとおりです。

- npm: パッケージ名
- git: ref を除いたリポジトリ URL
- local: 解決済み絶対パス
