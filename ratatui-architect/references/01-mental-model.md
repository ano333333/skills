# Mental Model

## Immediate Mode を前提に考える

`ratatui` の UI は、保持された widget 木をあとから更新するモデルではない。各 frame で、現在の state から必要な UI を描き直す。

この前提から、自然な設計上の結論がいくつか出る。

## 設計上の帰結

- widget を永続的な owner として持たない
- render 関数は「現在の state の投影」として扱う
- visual change を起こしたいなら state を更新し、その結果として再描画する
- widget の都合ではなく state の都合で設計する

## Retained Mode との違い

retained mode 的な発想:

- widget を生成して保持する
- widget に値を後から流し込む
- widget 間の依存を widget tree で表す

`ratatui` で自然な発想:

- `App` や component state を保持する
- 毎 frame、現在 state に基づいて widget を組み立てる
- 依存関係は state と event flow で表す

## Render 関数の責務

render は以下に絞る。

- `Frame` を受け取る
- `Layout` で領域を切る
- state を読み、必要な widget をそこへ描画する

render の中で避けるべきもの:

- ネットワーク I/O
- ファイル読み込み
- 重い変換
- 複雑な分岐を伴う business logic

## 自然なコードの形

- `update` が state を変える
- `view` / `render` が state を描く
- event loop が `update` と `draw` をつなぐ

この分離が崩れると、`ratatui` ではすぐに読みづらくなる。

