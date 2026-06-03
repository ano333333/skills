# Async And Background Work

`ratatui` の render loop に I/O を混ぜると破綻しやすい。非同期処理は event source の一つとして扱う。

## 原則

- draw は速く保つ
- blocking I/O は別 task / thread へ逃がす
- task 完了通知を action として main loop へ戻す

## 自然な構成

- input reader
- tick producer
- background worker
- main update loop
- renderer

## よくある流れ

1. ユーザー操作が action を生む
2. action が side effect を要求する
3. worker が I/O を実行する
4. 完了結果を action として戻す
5. state 更新後に再描画する

## 向いているケース

- tail するログビューア
- サーバ監視ダッシュボード
- 検索結果を非同期ロードする explorer
- 長時間実行ジョブの進捗表示

## 注意点

- render loop と task lifecycle を密結合させない
- task ハンドルを UI component に閉じ込めすぎない
- background result をそのまま widget に流さず state に正規化する

