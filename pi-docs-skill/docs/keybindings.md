# キーバインド

> 要約: Pi のキーボードショートカットは `~/.pi/agent/keybindings.json` で全面的にカスタマイズできます。このページでは、既定アクション一覧、キー表記ルール、Windows/WSL を含む注意点、設定例をまとめています。

すべてのキーボードショートカットは `~/.pi/agent/keybindings.json` でカスタマイズできます。各アクションには 1 つ以上のキーを割り当てられます。

設定ファイルでは、Pi 内部で使われる名前空間付きキーバインド ID と、拡張作者が `keyHint()` や注入された `keybindings` マネージャで使う ID と同じものを使用します。

`cursorUp` や `expandTools` のような旧来の名前空間なし ID を使った古い設定は、起動時に自動的に名前空間付き ID へ移行されます。

`keybindings.json` を編集したあとは、セッションを再起動せずに反映するため `pi` 内で `/reload` を実行してください。

## キー形式

`modifier+key` 形式です。modifier には `ctrl`、`shift`、`alt` を使い、組み合わせも可能です。key には次を使います。

- **文字:** `a-z`
- **数字:** `0-9`
- **特殊キー:** `escape`、`esc`、`enter`、`return`、`tab`、`space`、`backspace`、`delete`、`insert`、`clear`、`home`、`end`、`pageUp`、`pageDown`、`up`、`down`、`left`、`right`
- **ファンクションキー:** `f1`-`f12`
- **記号:** `` ` ``、`-`、`=`、`[`、`]`、`\`、`;`、`'`、`,`、`.`、`/`、`!`、`@`、`#`、`$`、`%`、`^`、`&`、`*`、`(`、`)`、`_`、`+`、`|`、`~`、`{`、`}`、`:`、`<`、`>`、`?`

modifier の組み合わせ例: `ctrl+shift+x`、`alt+ctrl+x`、`ctrl+shift+alt+x`、`ctrl+1` など。

## すべてのアクション

### TUI エディタのカーソル移動

| Keybinding id | 既定値 | 説明 |
|--------|---------|-------------|
| `tui.editor.cursorUp` | `up` | カーソルを上へ移動 |
| `tui.editor.cursorDown` | `down` | カーソルを下へ移動 |
| `tui.editor.cursorLeft` | `left`, `ctrl+b` | カーソルを左へ移動 |
| `tui.editor.cursorRight` | `right`, `ctrl+f` | カーソルを右へ移動 |
| `tui.editor.cursorWordLeft` | `alt+left`, `ctrl+left`, `alt+b` | 単語単位で左へ移動 |
| `tui.editor.cursorWordRight` | `alt+right`, `ctrl+right`, `alt+f` | 単語単位で右へ移動 |
| `tui.editor.cursorLineStart` | `home`, `ctrl+a` | 行頭へ移動 |
| `tui.editor.cursorLineEnd` | `end`, `ctrl+e` | 行末へ移動 |
| `tui.editor.jumpForward` | `ctrl+]` | 文字を指定して前方へジャンプ |
| `tui.editor.jumpBackward` | `ctrl+alt+]` | 文字を指定して後方へジャンプ |
| `tui.editor.pageUp` | `pageUp` | 1 ページ分上へスクロール |
| `tui.editor.pageDown` | `pageDown` | 1 ページ分下へスクロール |

### TUI エディタの削除操作

| Keybinding id | 既定値 | 説明 |
|--------|---------|-------------|
| `tui.editor.deleteCharBackward` | `backspace` | 後方の 1 文字を削除 |
| `tui.editor.deleteCharForward` | `delete`, `ctrl+d` | 前方の 1 文字を削除 |
| `tui.editor.deleteWordBackward` | `ctrl+w`, `alt+backspace` | 後方の 1 単語を削除 |
| `tui.editor.deleteWordForward` | `alt+d`, `alt+delete` | 前方の 1 単語を削除 |
| `tui.editor.deleteToLineStart` | `ctrl+u` | 行頭まで削除 |
| `tui.editor.deleteToLineEnd` | `ctrl+k` | 行末まで削除 |

### TUI 入力

| Keybinding id | 既定値 | 説明 |
|--------|---------|-------------|
| `tui.input.newLine` | `shift+enter` | 改行を挿入 |
| `tui.input.submit` | `enter` | 入力を送信 |
| `tui.input.tab` | `tab` | Tab / オートコンプリート |

### TUI Kill Ring

| Keybinding id | 既定値 | 説明 |
|--------|---------|-------------|
| `tui.editor.yank` | `ctrl+y` | 直近で削除したテキストを貼り付け |
| `tui.editor.yankPop` | `alt+y` | yank 後に削除履歴を巡回 |
| `tui.editor.undo` | `ctrl+-` | 最後の編集を取り消し |

### TUI クリップボードと選択

| Keybinding id | 既定値 | 説明 |
|--------|---------|-------------|
| `tui.input.copy` | `ctrl+c` | 選択範囲をコピー |
| `tui.select.up` | `up` | 選択を上へ移動 |
| `tui.select.down` | `down` | 選択を下へ移動 |
| `tui.select.pageUp` | `pageUp` | リスト内で 1 ページ上へ |
| `tui.select.pageDown` | `pageDown` | リスト内で 1 ページ下へ |
| `tui.select.confirm` | `enter` | 選択を確定 |
| `tui.select.cancel` | `escape`, `ctrl+c` | 選択をキャンセル |

### アプリケーション

| Keybinding id | 既定値 | 説明 |
|--------|---------|-------------|
| `app.interrupt` | `escape` | キャンセル / 中断 |
| `app.clear` | `ctrl+c` | エディタをクリア |
| `app.exit` | `ctrl+d` | 終了（エディタが空のとき） |
| `app.suspend` | `ctrl+z` (Windows ではなし) | バックグラウンドへサスペンド |
| `app.editor.external` | `ctrl+g` | 外部エディタ (`$VISUAL` または `$EDITOR`) で開く |
| `app.clipboard.pasteImage` | `ctrl+v` (Windows では `alt+v`) | クリップボードから画像を貼り付け |

### セッション

| Keybinding id | 既定値 | 説明 |
|--------|---------|-------------|
| `app.session.new` | *(なし)* | 新しいセッションを開始 (`/new`) |
| `app.session.tree` | *(なし)* | セッションツリーを開く (`/tree`) |
| `app.session.resume` | *(なし)* | セッション再開ピッカーを開く (`/resume`) |
| `app.session.togglePath` | `ctrl+p` | パス表示を切り替え |
| `app.session.toggleSort` | `ctrl+s` | ソートモードを切り替え |
| `app.session.toggleNamedFilter` | `ctrl+n` | 名前付きのみフィルタを切り替え |
| `app.session.rename` | `ctrl+r` | セッション名を変更 |
| `app.session.delete` | `ctrl+d` | セッションを削除 |
| `app.session.deleteNoninvasive` | `ctrl+backspace` | クエリが空のときにセッションを削除 |

### モデルと思考

| Keybinding id | 既定値 | 説明 |
|--------|---------|-------------|
| `app.model.select` | `ctrl+l` | モデルセレクタを開く |
| `app.model.cycleForward` | `ctrl+p` | 次のモデルへ切り替え |
| `app.model.cycleBackward` | `shift+ctrl+p` | 前のモデルへ切り替え |
| `app.thinking.cycle` | `shift+tab` | 思考レベルを巡回 |
| `app.thinking.toggle` | `ctrl+t` | thinking ブロックを折りたたみ/展開 |

### 表示とメッセージキュー

| Keybinding id | 既定値 | 説明 |
|--------|---------|-------------|
| `app.tools.expand` | `ctrl+o` | ツール出力を折りたたみ/展開 |
| `app.message.followUp` | `alt+enter` | follow-up メッセージをキュー |
| `app.message.dequeue` | `alt+up` | キュー済みメッセージをエディタへ戻す |

### ツリーナビゲーション

| Keybinding id | 既定値 | 説明 |
|--------|---------|-------------|
| `app.tree.foldOrUp` | `ctrl+left`, `alt+left` | 現在のブランチ区間を折りたたむ、または前の区間先頭へ移動 |
| `app.tree.unfoldOrDown` | `ctrl+right`, `alt+right` | 現在のブランチ区間を展開する、または次の区間先頭/ブランチ終端へ移動 |
| `app.tree.editLabel` | `shift+l` | 選択中ツリーノードのラベルを編集 |
| `app.tree.toggleLabelTimestamp` | `shift+t` | ツリー内のラベル時刻表示を切り替え |
| `app.tree.filter.default` | `ctrl+d` | ツリーフィルタを既定表示にする |
| `app.tree.filter.noTools` | `ctrl+t` | ツール結果を隠すフィルタを切り替え |
| `app.tree.filter.userOnly` | `ctrl+u` | ユーザーメッセージのみ表示するフィルタを切り替え |
| `app.tree.filter.labeledOnly` | `ctrl+l` | ラベル付きエントリのみ表示するフィルタを切り替え |
| `app.tree.filter.all` | `ctrl+a` | 全エントリ表示フィルタを切り替え |
| `app.tree.filter.cycleForward` | `ctrl+o` | ツリーフィルタを順方向に巡回 |
| `app.tree.filter.cycleBackward` | `shift+ctrl+o` | ツリーフィルタを逆方向に巡回 |

### Scoped Models Selector

scoped models selector（`/scoped-models` から開く）内で使います。

| Keybinding id | 既定値 | 説明 |
|--------|---------|-------------|
| `app.models.save` | `ctrl+s` | 現在のモデル選択を設定へ保存 |
| `app.models.enableAll` | `ctrl+a` | すべてのモデルを有効化（または検索一致分のみ） |
| `app.models.clearAll` | `ctrl+x` | すべてのモデルをクリア（または検索一致分のみ） |
| `app.models.toggleProvider` | `ctrl+p` | 現在のプロバイダ配下の全モデルを切り替え |
| `app.models.reorderUp` | `alt+up` | 選択モデルを循環順序の上へ移動 |
| `app.models.reorderDown` | `alt+down` | 選択モデルを循環順序の下へ移動 |

## カスタム設定

`~/.pi/agent/keybindings.json` を作成します。

```json
{
  "tui.editor.cursorUp": ["up", "ctrl+p"],
  "tui.editor.cursorDown": ["down", "ctrl+n"],
  "tui.editor.deleteWordBackward": ["ctrl+w", "alt+backspace"]
}
```

各アクションには 1 つのキーまたはキー配列を指定できます。ユーザー設定は既定値を上書きします。

ネイティブ Windows では、Windows 端末が Unix のジョブ制御をサポートしないため、`app.suspend` に既定の割り当てはありません。手動で割り当てた場合も、Pi はサスペンドせずステータスメッセージを表示します。WSL では通常の Linux と同じ `ctrl+z` / `fg` の挙動が引き続き使えます。

### Emacs 例

```json
{
  "tui.editor.cursorUp": ["up", "ctrl+p"],
  "tui.editor.cursorDown": ["down", "ctrl+n"],
  "tui.editor.cursorLeft": ["left", "ctrl+b"],
  "tui.editor.cursorRight": ["right", "ctrl+f"],
  "tui.editor.cursorWordLeft": ["alt+left", "alt+b"],
  "tui.editor.cursorWordRight": ["alt+right", "alt+f"],
  "tui.editor.deleteCharForward": ["delete", "ctrl+d"],
  "tui.editor.deleteCharBackward": ["backspace", "ctrl+h"],
  "tui.input.newLine": ["shift+enter", "ctrl+j"]
}
```

### Vim 例

```json
{
  "tui.editor.cursorUp": ["up", "alt+k"],
  "tui.editor.cursorDown": ["down", "alt+j"],
  "tui.editor.cursorLeft": ["left", "alt+h"],
  "tui.editor.cursorRight": ["right", "alt+l"],
  "tui.editor.cursorWordLeft": ["alt+left", "alt+b"],
  "tui.editor.cursorWordRight": ["alt+right", "alt+w"]
}
```
