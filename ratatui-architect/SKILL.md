---
name: ratatui-architect
description: Ratatui で Rust の TUI アプリを設計・実装するときの設計支援スキル。`ratatui` を使ったダッシュボード、ログビューア、エディタ、フォーム、マルチペイン UI、非同期イベント駆動 UI を作る相談では必ず使う。struct や fn の列挙ではなく、immediate mode rendering、state の置き方、event loop、layout 分割、component / TEA / Flux の選び分け、自然な module 構成を優先して案内する。
---

# Ratatui Architect

`ratatui` の公式 docs を、API 索引ではなく「どう考えて組むと自然か」の観点で再編したスキルです。

## What This Skill Optimizes For

- API 名の丸暗記よりも、`ratatui` の前提に沿った自然な設計を優先する
- immediate mode rendering を出発点に、state / event / rendering の責務分離を行う
- 小さな単画面から複雑な非同期 UI まで、規模に合った構成を選ぶ
- 回答はまず設計方針、その後に module 構成、最後に必要な API 逆引きの順で示す

## Default Workflow

1. まずユーザーが作りたいアプリの shape を判定する
2. state の種類を切り分ける
3. event loop と action flow を決める
4. layout を widget より先に設計する
5. 最後に必要最小限の `ratatui` API へ落とす

## First Questions To Answer

以下を短く整理してから設計を始める:

- 単画面か、多画面か
- 入力中心か、表示中心か
- 同期 loop で足りるか、非同期イベントが多いか
- 選択状態、スクロール状態、カーソル状態をどこに置くか
- 画面ごとの責務を component に切るべきか、単一 `App` で足りるか

## Response Shape

ユーザーへの回答は原則として以下の順で組み立てる:

1. 設計上の前提
2. 推奨アーキテクチャ
3. module / type の分割案
4. event と state の流れ
5. 必要なら API 名の最小セット

`ratatui` では API 名よりも責務配置の方が重要なので、struct や fn の説明は補助として扱う。

## Reference Routing

- 全体像から入りたい: `references/00-start-here.md`
- immediate mode の考え方を押さえたい: `references/01-mental-model.md`
- architecture を選びたい: `references/02-app-shapes-and-architecture.md`
- layout 分割を考えたい: `references/03-layout-and-composition.md`
- state / event / action の流れを固めたい: `references/04-state-events-and-actions.md`
- widget の使い分けや custom widget を考えたい: `references/05-widget-usage-and-customization.md`
- async / background work を扱いたい: `references/06-async-and-background-work.md`
- よくある構成と避けるべき罠を見たい: `references/07-common-patterns-and-anti-patterns.md`
- 設計から API を逆引きしたい: `references/08-api-wayfinding.md`

## Working Rules

- retained mode GUI の発想をそのまま持ち込まない
- widget tree を永続オブジェクトとして保持する前提で話さない
- render 中に I/O や重い処理を持ち込まない
- `Layout` と `Rect` を先に決め、widget 選定はその後に行う
- `StatefulWidget` の state は widget 自身ではなく app 側の state として扱う
- 大規模 UI では key handling を巨大な `match` 一つに押し込まない

## Suggested Answer Style

- まず「なぜその構成が `ratatui` に自然か」を説明する
- 次に Rust の module 構成を具体化する
- 最後に `Terminal::draw`, `Frame`, `Layout`, `render_stateful_widget` など必要な API だけ挙げる
- 実装例を出す場合でも、単なるサンプルコード列挙ではなく責務の境界が見える形にする

## Sources

- `https://docs.rs/ratatui/latest/ratatui/`
- `https://ratatui.rs/concepts/rendering/`
- `https://ratatui.rs/concepts/layout/`
- `https://ratatui.rs/concepts/widgets/`
- `https://ratatui.rs/concepts/event-handling/`
- `https://ratatui.rs/concepts/application-patterns/`

