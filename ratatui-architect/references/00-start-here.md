# Start Here

この skill は `ratatui` を「API 集」ではなく「設計制約を持つ描画ライブラリ」として扱います。

## Ratatui を一言でいうと

- `ratatui` は terminal backend の上で immediate mode rendering を行う UI ライブラリ
- 描画は助けてくれるが、event loop と architecture は利用側が決める
- したがって、最初に API を調べるより、state と loop の設計を決める方が先

## 先に決めること

1. 何を state として保持するか
2. どの event が state を変えるか
3. どの粒度で render を走らせるか
4. layout をどの段階で分割するか
5. 画面や機能を component に切るか

## 設計の基本順序

1. ドメイン state を決める
2. UI state を決める
3. event source を列挙する
4. action / message の流れを決める
5. layout を設計する
6. 最後に widget を当てる

## UI State と Domain State

分けて考えると設計が安定する。

- domain state:
  - 取得したログ
  - テーブルの行データ
  - フォーム値
  - 接続状態
- UI state:
  - 選択中の index
  - scroll offset
  - active pane
  - modal open/close
  - cursor position

`ratatui` では widget が長生きするのではなく state が長生きする。

## 次に読むべきもの

- immediate mode の違いが曖昧なら `01-mental-model.md`
- app の切り方に迷うなら `02-app-shapes-and-architecture.md`
- event loop をどう組むか迷うなら `04-state-events-and-actions.md`

