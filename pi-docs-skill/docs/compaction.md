# コンパクションとブランチ要約

> 要約: Pi は長い会話でコンテキスト枠を節約するためにコンパクションを行い、`/tree` で分岐を移動する際にはブランチ要約で文脈を引き継げます。どちらも同じ構造化要約フォーマットと累積的なファイル追跡を使い、拡張機能からのカスタマイズにも対応しています。

LLM には限られたコンテキストウィンドウがあります。会話が長くなりすぎると、pi は最近の作業を保持したまま古い内容を要約するためにコンパクションを使います。このページでは、自動コンパクションとブランチ要約の両方を扱います。

**ソースファイル**（[pi-mono](https://github.com/earendil-works/pi-mono)）:
- [`packages/coding-agent/src/core/compaction/compaction.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/compaction/compaction.ts) - 自動コンパクションのロジック
- [`packages/coding-agent/src/core/compaction/branch-summarization.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/compaction/branch-summarization.ts) - ブランチ要約
- [`packages/coding-agent/src/core/compaction/utils.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/compaction/utils.ts) - 共通ユーティリティ（ファイル追跡、シリアライズ）
- [`packages/coding-agent/src/core/session-manager.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/session-manager.ts) - エントリ型（`CompactionEntry`, `BranchSummaryEntry`）
- [`packages/coding-agent/src/core/extensions/types.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/extensions/types.ts) - 拡張機能イベント型

プロジェクト内の TypeScript 定義は、`node_modules/@earendil-works/pi-coding-agent/dist/` を確認してください。

## 概要

Pi には 2 つの要約メカニズムがあります。

| メカニズム | 発火条件 | 目的 |
|-----------|---------|---------|
| コンパクション | コンテキストがしきい値を超える、または `/compact` | 古いメッセージを要約してコンテキストを解放する |
| ブランチ要約 | `/tree` ナビゲーション | ブランチ切り替え時に文脈を保持する |

どちらも同じ構造化要約フォーマットを使い、ファイル操作を累積的に追跡します。

## コンパクション

### 発火するタイミング

自動コンパクションは次の場合に発火します。

```
contextTokens > contextWindow - reserveTokens
```

デフォルトでは `reserveTokens` は 16384 トークンです（`~/.pi/agent/settings.json` または `<project-dir>/.pi/settings.json` で設定可能）。これは LLM の応答用の余地を残すためです。

また、`/compact [instructions]` で手動実行することもできます。任意の instructions を付けると、要約の焦点を指定できます。

### 仕組み

1. **切断点を見つける**: 最新メッセージから逆方向にたどり、`keepRecentTokens`（デフォルト 20k、`~/.pi/agent/settings.json` または `<project-dir>/.pi/settings.json` で設定可能）に達するまでトークン推定値を加算する
2. **メッセージを抽出する**: 直前の保持境界（またはセッション開始）から切断点までのメッセージを集める
3. **要約を生成する**: 構造化フォーマットで要約するために LLM を呼び出し、前回の要約があれば反復的な文脈として渡す
4. **エントリを追加する**: 要約と `firstKeptEntryId` を持つ `CompactionEntry` を保存する
5. **再読み込みする**: セッションは再読み込みされ、要約と `firstKeptEntryId` 以降のメッセージを使う

```
コンパクション前:

  entry:  0     1     2     3      4     5     6      7      8     9
        ┌─────┬─────┬─────┬─────┬──────┬─────┬─────┬──────┬──────┬─────┐
        │ hdr │ usr │ ass │ tool │ usr │ ass │ tool │ tool │ ass │ tool│
        └─────┴─────┴─────┴──────┴─────┴─────┴──────┴──────┴─────┴─────┘
                └────────┬───────┘ └──────────────┬──────────────┘
               messagesToSummarize            kept messages
                                   ↑
                          firstKeptEntryId (entry 4)

コンパクション後（新しいエントリを追加）:

  entry:  0     1     2     3      4     5     6      7      8     9     10
        ┌─────┬─────┬─────┬─────┬──────┬─────┬─────┬──────┬──────┬─────┬─────┐
        │ hdr │ usr │ ass │ tool │ usr │ ass │ tool │ tool │ ass │ tool│ cmp │
        └─────┴─────┴─────┴──────┴─────┴─────┴──────┴──────┴─────┴─────┴─────┘
               └──────────┬──────┘ └──────────────────────┬───────────────────┘
                 not sent to LLM                    sent to LLM
                                                         ↑
                                              starts from firstKeptEntryId

LLM が見るもの:

  ┌────────┬─────────┬─────┬─────┬──────┬──────┬─────┬──────┐
  │ system │ summary │ usr │ ass │ tool │ tool │ ass │ tool │
  └────────┴─────────┴─────┴─────┴──────┴──────┴─────┴──────┘
       ↑         ↑      └─────────────────┬────────────────┘
    prompt   from cmp          messages from firstKeptEntryId
```

繰り返しコンパクションが行われる場合、要約対象の開始位置はコンパクションエントリ自身ではなく、前回コンパクションの保持境界（`firstKeptEntryId`）になります。もしその保持エントリが経路上で見つからなければ、前回コンパクション直後のエントリにフォールバックします。これにより、前回のコンパクションを生き残ったメッセージも次回の要約処理に再び含められます。Pi は新しい `CompactionEntry` を書き込む前に、再構築したセッションコンテキストから `tokensBefore` も再計算するため、このトークン数は実際に置き換えられるコンパクション前コンテキストを正しく反映します。

### 分割ターン

「ターン」はユーザーメッセージから始まり、次のユーザーメッセージが来るまでのすべてのアシスタント応答とツール呼び出しを含みます。通常、コンパクションはターン境界で切れます。

1 回のターンが `keepRecentTokens` を超える場合、切断点はターンの途中にあるアシスタントメッセージに着地します。これを「分割ターン」と呼びます。

```
分割ターン（巨大な 1 ターンが予算を超える場合）:

  entry:  0     1     2      3     4      5      6     7      8
        ┌─────┬─────┬─────┬──────┬─────┬──────┬──────┬─────┬──────┐
        │ hdr │ usr │ ass │ tool │ ass │ tool │ tool │ ass │ tool │
        └─────┴─────┴─────┴──────┴─────┴──────┴──────┴─────┴──────┘
                ↑                                     ↑
         turnStartIndex = 1                  firstKeptEntryId = 7
                │                                     │
                └──── turnPrefixMessages (1-6) ───────┘
                                                      └── kept (7-8)

  isSplitTurn = true
  messagesToSummarize = []  (その前に完全なターンはない)
  turnPrefixMessages = [usr, ass, tool, ass, tool, tool]
```

分割ターンでは、pi は 2 つの要約を生成して結合します。
1. **履歴要約**: 以前のコンテキスト（存在する場合）
2. **ターン接頭部要約**: 分割されたターン前半部分

### 切断点のルール

有効な切断点は次のとおりです。
- ユーザーメッセージ
- アシスタントメッセージ
- BashExecution メッセージ
- カスタムメッセージ（custom_message, branch_summary）

ツール結果では絶対に切断しません（ツール呼び出しと一緒に残す必要があるためです）。

### CompactionEntry の構造

[`session-manager.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/session-manager.ts) で定義されています。

```typescript
interface CompactionEntry<T = unknown> {
  type: "compaction";
  id: string;
  parentId: string;
  timestamp: number;
  summary: string;
  firstKeptEntryId: string;
  tokensBefore: number;
  fromHook?: boolean;  // 拡張機能が提供した場合は true（レガシーなフィールド名）
  details?: T;         // 実装固有のデータ
}

// デフォルトのコンパクションでは details にこれを使う（compaction.ts より）:
interface CompactionDetails {
  readFiles: string[];
  modifiedFiles: string[];
}
```

拡張機能は `details` に JSON シリアライズ可能な任意のデータを保存できます。デフォルトのコンパクションはファイル操作を追跡しますが、カスタム拡張実装では独自の構造を使えます。

実装は [`prepareCompaction()`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/compaction/compaction.ts) と [`compact()`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/compaction/compaction.ts) を参照してください。

## ブランチ要約

### 発火するタイミング

`/tree` を使って別のブランチへ移動すると、pi は離れる側の作業を要約するかどうかを提案します。これにより、離れるブランチの文脈を新しいブランチへ注入できます。

### 仕組み

1. **共通祖先を見つける**: 古い位置と新しい位置が共有するもっとも深いノード
2. **エントリを集める**: 古い leaf から共通祖先まで逆向きにたどる
3. **予算付きで準備する**: トークン予算に収まるメッセージを含める（新しい順）
4. **要約を生成する**: 構造化フォーマットで LLM を呼び出す
5. **エントリを追加する**: ナビゲーション位置に `BranchSummaryEntry` を保存する

```
ナビゲーション前のツリー:

         ┌─ B ─ C ─ D (old leaf, being abandoned)
    A ───┤
         └─ E ─ F (target)

共通祖先: A
要約対象のエントリ: B, C, D

要約付きナビゲーション後:

         ┌─ B ─ C ─ D ─ [summary of B,C,D]
    A ───┤
         └─ E ─ F (new leaf)
```

### 累積的なファイル追跡

コンパクションとブランチ要約はどちらもファイルを累積的に追跡します。要約を生成するとき、pi は次からファイル操作を抽出します。
- 要約対象メッセージ内のツール呼び出し
- 以前の compaction または branch summary の `details`（あれば）

つまり、ファイル追跡は複数回のコンパクションや入れ子になったブランチ要約をまたいで蓄積され、読み取ったファイルと変更したファイルの完全な履歴を保持できます。

### BranchSummaryEntry の構造

[`session-manager.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/session-manager.ts) で定義されています。

```typescript
interface BranchSummaryEntry<T = unknown> {
  type: "branch_summary";
  id: string;
  parentId: string;
  timestamp: number;
  summary: string;
  fromId: string;      // 移動元のエントリ
  fromHook?: boolean;  // 拡張機能が提供した場合は true（レガシーなフィールド名）
  details?: T;         // 実装固有のデータ
}

// デフォルトのブランチ要約では details にこれを使う（branch-summarization.ts より）:
interface BranchSummaryDetails {
  readFiles: string[];
  modifiedFiles: string[];
}
```

コンパクションと同様に、拡張機能は `details` にカスタムデータを保存できます。

実装は [`collectEntriesForBranchSummary()`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/compaction/branch-summarization.ts)、[`prepareBranchEntries()`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/compaction/branch-summarization.ts)、[`generateBranchSummary()`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/compaction/branch-summarization.ts) を参照してください。

## 要約フォーマット

コンパクションとブランチ要約はどちらも同じ構造化フォーマットを使います。

```markdown
## Goal
[ユーザーが達成しようとしていること]

## Constraints & Preferences
- [ユーザーが述べた要件]

## Progress
### Done
- [x] [完了したタスク]

### In Progress
- [ ] [現在の作業]

### Blocked
- [問題があれば記載]

## Key Decisions
- **[決定事項]**: [理由]

## Next Steps
1. [次に起きるべきこと]

## Critical Context
- [継続に必要なデータ]

<read-files>
path/to/file1.ts
path/to/file2.ts
</read-files>

<modified-files>
path/to/changed.ts
</modified-files>
```

### メッセージのシリアライズ

要約の前に、メッセージは [`serializeConversation()`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/compaction/utils.ts) によってテキストへシリアライズされます。

```
[User]: 発言内容
[Assistant thinking]: 内部推論
[Assistant]: 応答テキスト
[Assistant tool calls]: read(path="foo.ts"); edit(path="bar.ts", ...)
[Tool result]: ツールからの出力
```

これにより、モデルがそれを継続すべき会話として扱うのを防げます。

ツール結果はシリアライズ時に 2000 文字へ切り詰められます。上限を超えた部分は、何文字切り詰めたかを示すマーカーに置き換えられます。これにより、要約リクエストが妥当なトークン予算内に収まります。特に `read` や `bash` の結果はコンテキストサイズへの寄与がもっとも大きくなりやすいためです。

## 拡張機能によるカスタム要約

拡張機能はコンパクションとブランチ要約の両方を横取りしてカスタマイズできます。イベント型定義は [`extensions/types.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/extensions/types.ts) を参照してください。

### session_before_compact

自動コンパクションまたは `/compact` の前に発火します。キャンセルしたり、カスタム要約を提供したりできます。型ファイル内の `SessionBeforeCompactEvent` と `CompactionPreparation` を参照してください。

```typescript
pi.on("session_before_compact", async (event, ctx) => {
  const { preparation, branchEntries, customInstructions, signal } = event;

  // preparation.messagesToSummarize - 要約対象のメッセージ
  // preparation.turnPrefixMessages - 分割ターンの接頭部（isSplitTurn の場合）
  // preparation.previousSummary - 前回のコンパクション要約
  // preparation.fileOps - 抽出されたファイル操作
  // preparation.tokensBefore - コンパクション前のコンテキストトークン数
  // preparation.firstKeptEntryId - 保持メッセージの開始位置
  // preparation.settings - コンパクション設定

  // branchEntries - 現在のブランチ上の全エントリ（カスタム状態用）
  // signal - AbortSignal（LLM 呼び出しに渡す）

  // キャンセル:
  return { cancel: true };

  // カスタム要約:
  return {
    compaction: {
      summary: "Your summary...",
      firstKeptEntryId: preparation.firstKeptEntryId,
      tokensBefore: preparation.tokensBefore,
      details: { /* custom data */ },
    }
  };
});
```

#### メッセージをテキストへ変換する

独自モデルで要約を生成するには、`serializeConversation` を使ってメッセージをテキストへ変換します。

```typescript
import { convertToLlm, serializeConversation } from "@earendil-works/pi-coding-agent";

pi.on("session_before_compact", async (event, ctx) => {
  const { preparation } = event;
  
  // AgentMessage[] を Message[] に変換し、その後テキストへシリアライズする
  const conversationText = serializeConversation(
    convertToLlm(preparation.messagesToSummarize)
  );
  // 戻り値:
  // [User]: メッセージ本文
  // [Assistant thinking]: thinking の内容
  // [Assistant]: 応答本文
  // [Assistant tool calls]: read(path="..."); bash(command="...")
  // [Tool result]: 出力本文

  // ここから独自モデルへ送って要約する
  const summary = await myModel.summarize(conversationText);
  
  return {
    compaction: {
      summary,
      firstKeptEntryId: preparation.firstKeptEntryId,
      tokensBefore: preparation.tokensBefore,
    }
  };
});
```

別モデルを使う完全な例は [custom-compaction.ts](../examples/extensions/custom-compaction.ts) を参照してください。

### session_before_tree

`/tree` ナビゲーションの前に発火します。ユーザーが要約を選んだかどうかに関係なく常に発火します。ナビゲーション全体をキャンセルしたり、カスタム要約を提供したりできます。

```typescript
pi.on("session_before_tree", async (event, ctx) => {
  const { preparation, signal } = event;

  // preparation.targetId - 移動先
  // preparation.oldLeafId - 現在位置（離脱される側）
  // preparation.commonAncestorId - 共通祖先
  // preparation.entriesToSummarize - 要約対象になるエントリ
  // preparation.userWantsSummary - ユーザーが要約を選んだかどうか

  // ナビゲーション全体をキャンセル:
  return { cancel: true };

  // カスタム要約を提供（userWantsSummary が true の場合のみ使用される）:
  if (preparation.userWantsSummary) {
    return {
      summary: {
        summary: "Your summary...",
        details: { /* custom data */ },
      }
    };
  }
});
```

型ファイルの `SessionBeforeTreeEvent` と `TreePreparation` を参照してください。

## 設定

`~/.pi/agent/settings.json` または `<project-dir>/.pi/settings.json` でコンパクションを設定します。

```json
{
  "compaction": {
    "enabled": true,
    "reserveTokens": 16384,
    "keepRecentTokens": 20000
  }
}
```

| 設定 | デフォルト | 説明 |
|---------|---------|-------------|
| `enabled` | `true` | 自動コンパクションを有効にする |
| `reserveTokens` | `16384` | LLM 応答用に予約するトークン数 |
| `keepRecentTokens` | `20000` | 保持する最近のトークン数（要約しない） |

`"enabled": false` で自動コンパクションを無効にできます。その場合でも `/compact` による手動コンパクションは使えます。
