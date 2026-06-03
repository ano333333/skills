# Common Patterns And Anti-Patterns

## よくある自然なアプリ形

### Dashboard

- 中央 state
- 上部 summary
- 中央 main pane
- 下部 status / logs
- tick または async action で更新

### List-Detail Browser

- list selection state
- detail pane は選択 item から再計算
- key handling は list action と global action を分ける

### Modal Form

- form state
- focus state
- validation result
- modal open/close は app level state

### Log Viewer

- append-only domain data
- filter query
- scroll / follow mode
- background ingestion と UI state を分離

## Anti-Patterns

### 巨大な `App` にすべてを詰め込む

短期では速いが、pane と keybinding が増えると破綻しやすい。

### key event を意味変換せず直接処理する

入力装置依存のままロジックに入るので、設定変更やテストがつらい。

### widget を永続 state の owner にする

`ratatui` の mental model と衝突する。

### render 中にデータ取得する

フレーム落ちと責務混線の原因になる。

### domain state と UI state を混ぜる

選択状態や modal 状態が business object に侵食しやすい。

