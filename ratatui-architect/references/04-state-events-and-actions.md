# State, Events, And Actions

`ratatui` は event handling を直接提供しない。ここをどう組むかで保守性が大きく変わる。

## Event Source を分ける

- keyboard input
- mouse input
- tick
- network / file / background worker 完了通知
- resize

## 推奨する流れ

### 小規模

`Event -> App::handle_event -> mutate state -> draw`

### 中規模

`Event -> translate to Action/Message -> route -> mutate state -> draw`

### 非同期あり

`Input/Task Result -> channel -> Action -> reducer/update -> effect trigger -> draw`

## 何を action にするか

action / message は「ユーザー意図」または「非同期結果」を表す単位にする。

よい例:

- `MoveSelectionDown`
- `OpenHelp`
- `LogsLoaded(Vec<LogLine>)`
- `SearchSubmitted(String)`

避けたい例:

- `PressedJ`
- `PressedCtrlC`

キー入力そのものより、意味のある action に変換した方が後で変えやすい。

## UI State の置き場所

- 全体に関わるものは `App`
- pane 固有のものは component state
- widget 描画に必要な selection / scroll state は対応 component に寄せる

## Resize の扱い

resize は domain state より render に閉じ込める方が自然なことが多い。画面サイズに応じた描画は layout で解決し、状態にしなくて済むならしない。

## Keybinding 設計

- 入力の生値処理は薄く保つ
- 意味解釈を action 変換層に寄せる
- component ごとの keymap を許容する

