# API Wayfinding

このファイルは item の網羅ではなく、設計判断から必要 API を逆引きするための補助です。

## 画面を領域分割したい

- `Layout`
- `Constraint`
- `Direction`
- `Rect`

## 毎 frame 描画したい

- `Terminal::draw`
- `Frame`
- `render_widget`
- `render_stateful_widget`

## 選択状態つきの一覧を描きたい

- `List`
- `ListState`
- `render_stateful_widget`

## table や複数列表示をしたい

- `Table`
- `Row`
- `Cell`
- `TableState`

## テキストや装飾を組みたい

- `Paragraph`
- `Line`
- `Span`
- `Style`
- `Block`
- `Borders`

## popup や枠つき pane を出したい

- `Block`
- 中央 `Rect` を作る layout helper
- 必要なら `Clear`

## 端末初期化と復元を整理したい

- crate docs の `Writing Applications`
- backend setup (`crossterm` など)
- terminal init / restore helper 群

## 重要な注意

API 名を探す前に、以下を言語化できるか確認する。

- state はどこにあるか
- どの action が state を変えるか
- render はどの単位で分かれるか
- layout はどう切られるか

ここが曖昧なまま API へ行くと、`ratatui` では実装がすぐ崩れる。

