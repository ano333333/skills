# Widget Usage And Customization

## Widget は View であって State の本体ではない

`ratatui` の built-in widget は、app state を terminal 上に投影するための view layer と考える。

## Stateless と Stateful

### Stateless Widget

向いている:

- `Paragraph`
- `Block`
- 単純な装飾

扱い:

- 毎 frame 普通に組み立てて描画する

### Stateful Widget

向いている:

- selection を持つ list
- table の行選択
- scroll や cursor を伴う表示

扱い:

- widget state は app 側に保持する
- render 時に `render_stateful_widget` へ渡す

## WidgetRef / StatefulWidgetRef の含意

参照ベース描画が必要な場面はあるが、重要なのは API 差分より責務分離。描画 API の形よりも、state がどこに住むかを先に決める。

## Custom Widget を書くべきとき

- 複数 widget の組み合わせを再利用したい
- 独自の描画アルゴリズムがある
- screen の一部を self-contained な view として切り出したい

## Custom Widget を書かなくてよいとき

- 単に `render_widget` を数回呼ぶだけで十分
- 再利用の境界がまだ見えていない
- stateful な振る舞いを widget 側に押し込めたいだけ

custom widget は view の抽象化であり、state 管理の逃げ道として使わない。

