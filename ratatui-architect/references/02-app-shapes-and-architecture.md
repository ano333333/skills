# App Shapes And Architecture

`ratatui` の公式 docs では、単一ループ、message passing、component architecture、TEA、Flux など複数の組み方が紹介されている。正解は一つではなく、規模と変更頻度で選ぶ。

## まずはこれで足りるかを判定する

### 1. Single App Struct

向いている:

- 単画面
- keybind が少ない
- modal が 1 つか 2 つ
- 非同期処理が少ない

構成:

- `App`
- `handle_event`
- `render`
- `run`

### 2. Component Architecture

向いている:

- pane や section が独立している
- 画面ごとに独自 keybind がある
- 再利用したい UI 片がある

構成:

- `app/`
- `components/`
- `screens/`
- 各 component が `render` と `handle_*` を持つ

### 3. TEA / Message-Driven

向いている:

- state 変化を一箇所に集約したい
- 画面遷移や modal 遷移が増える
- テストしやすい update ロジックが欲しい

構成:

- `Model`
- `Message`
- `update(model, msg)`
- `view(model, frame)`

### 4. Flux / Action Pipeline

向いている:

- 非同期イベント源が多い
- background task が多い
- UI action と side effect を分離したい

構成:

- `Action`
- `Store` または中央 state
- dispatcher / channel
- effect handler

## 選び方の目安

- 迷ったら `Single App Struct` から始める
- 複数 pane に独立した責務があるなら `Component Architecture`
- state transition が複雑なら `TEA`
- async と side effect が多いなら `Flux` 寄り

## Rust Module の切り方

小規模:

```text
src/
├── main.rs
├── app.rs
├── ui.rs
└── event.rs
```

中規模:

```text
src/
├── app/
├── components/
├── screens/
├── event.rs
├── action.rs
└── tui.rs
```

大規模:

```text
src/
├── model/
├── actions/
├── effects/
├── components/
├── screens/
├── routing/
└── tui/
```

