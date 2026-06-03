# Layout And Composition

`ratatui` では widget より `Layout` が先に来る。画面をどう切るかが、そのまま UI の設計図になる。

## 基本原則

- outer layout から inner layout へ順に切る
- 各 pane の責務を layout に表す
- `Rect` は render 時に得るものとして扱う
- 固定座標より constraint ベースを優先する

## 自然な分割順

1. 全体の縦横分割
2. header / body / footer
3. body 内の main pane / side pane
4. pane 内の widget 群

## 典型パターン

### List-Detail

- 左: 一覧
- 右上: 詳細ヘッダ
- 右下: 詳細本文

### Dashboard

- 上: サマリ
- 中: メインテーブル
- 下: ステータスやログ

### Modal Form

- 背景は通常 layout
- modal は中央 `Rect` を別計算

## `Rect` の扱い

原則として `Rect` を app state に保存しない方がよい。

理由:

- terminal size に依存する
- render 時にしか確定しない
- stale になりやすい

保存してよいのは、次 frame 以降に参照すべき UI 幾何がどうしても必要なときだけ。

## Anti-Pattern

- layout を後回しにして widget から積む
- business logic が `Rect` の存在に強く依存する
- pane ごとに別々の座標計算を手書きする

