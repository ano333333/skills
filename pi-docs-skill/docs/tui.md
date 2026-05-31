# TUI コンポーネント

> 要約: TUI コンポーネントは、拡張やカスタムツールから対話的な端末 UI を描画するための基盤です。入力処理、フォーカス、オーバーレイ、テーマ、描画幅、キャッシュ、無効化パターンまでを理解すると、実用的な UI を安全に組み立てられます。組み込み部品と推奨パターンも用意されています。

拡張やカスタムツールは、対話的なユーザーインターフェースのためにカスタム TUI コンポーネントを描画できます。このページでは、コンポーネントシステムと利用可能な構成要素を説明します。

**ソース:** [`@earendil-works/pi-tui`](https://github.com/earendil-works/pi-mono/tree/main/packages/tui)

## Component Interface

すべてのコンポーネントは次を実装します。

```typescript
interface Component {
  render(width: number): string[];
  handleInput?(data: string): void;
  wantsKeyRelease?: boolean;
  invalidate(): void;
}
```

| Method | Description |
|--------|-------------|
| `render(width)` | 文字列配列（1 行ごと）を返します。各行は **必ず `width` を超えてはいけません**。 |
| `handleInput?(data)` | コンポーネントがフォーカスを持つときにキーボード入力を受け取ります。 |
| `wantsKeyRelease?` | true の場合、コンポーネントはキーリリースイベント（Kitty プロトコル）を受け取ります。既定値は false です。 |
| `invalidate()` | キャッシュした描画状態をクリアします。テーマ変更時に呼ばれます。 |

TUI は各描画行の末尾に完全な SGR リセットと OSC 8 リセットを付加します。スタイルは行をまたいで保持されません。スタイル付きの複数行テキストを出力する場合は、行ごとにスタイルを再適用するか、`wrapTextWithAnsi()` を使って折り返し後の各行でもスタイルが保持されるようにしてください。

## Focusable Interface (IME Support)

テキストカーソルを表示し、IME（Input Method Editor）対応が必要なコンポーネントは、`Focusable` インターフェースを実装してください。

```typescript
import { CURSOR_MARKER, type Component, type Focusable } from "@earendil-works/pi-tui";

class MyInput implements Component, Focusable {
  focused: boolean = false;  // フォーカス変更時に TUI が設定する

  render(width: number): string[] {
    const marker = this.focused ? CURSOR_MARKER : "";
    // 擬似カーソルの直前にマーカーを出力する
    return [`> ${beforeCursor}${marker}\x1b[7m${atCursor}\x1b[27m${afterCursor}`];
  }
}
```

`Focusable` コンポーネントがフォーカスを持つと、TUI は以下を行います。
1. コンポーネントの `focused = true` を設定する
2. 描画出力から `CURSOR_MARKER`（幅を持たない APC エスケープシーケンス）を探す
3. その位置へ端末のハードウェアカーソルを移動する
4. `showHardwareCursor` が有効な場合のみハードウェアカーソルを表示する

カーソルはデフォルトでは非表示のままです。これにより、擬似カーソル描画を保ちつつ、カーソル非表示でも IME 候補ウィンドウを追従する端末向けにハードウェアカーソル位置を設定できます。一部の端末では IME 位置合わせに可視のハードウェアカーソルが必要です。その場合は `showHardwareCursor`、`setShowHardwareCursor(true)`、または `PI_HARDWARE_CURSOR=1` を使って有効化してください。組み込みの `Editor` と `Input` はすでにこのインターフェースを実装しています。

### Container Components with Embedded Inputs

コンテナコンポーネント（ダイアログ、セレクタなど）が `Input` や `Editor` の子を含む場合、コンテナ側で `Focusable` を実装し、フォーカス状態を子へ伝播する必要があります。そうしないと、IME 入力用のハードウェアカーソルが正しく配置されません。

```typescript
import { Container, type Focusable, Input } from "@earendil-works/pi-tui";

class SearchDialog extends Container implements Focusable {
  private searchInput: Input;

  // Focusable 実装: IME のカーソル位置合わせのために子 input へ伝播する
  private _focused = false;
  get focused(): boolean {
    return this._focused;
  }
  set focused(value: boolean) {
    this._focused = value;
    this.searchInput.focused = value;
  }

  constructor() {
    super();
    this.searchInput = new Input();
    this.addChild(this.searchInput);
  }
}
```

この伝播がないと、中国語、日本語、韓国語などの IME 入力時に候補ウィンドウが画面上の誤った位置に表示されます。

## Using Components

**拡張内**では `ctx.ui.custom()` を使います。

```typescript
pi.on("session_start", async (_event, ctx) => {
  const handle = ctx.ui.custom(myComponent);
  // handle.requestRender() - 再描画をトリガー
  // handle.close() - 通常 UI を復元
});
```

**カスタムツール内**では `pi.ui.custom()` を使います。

```typescript
async execute(toolCallId, params, onUpdate, ctx, signal) {
  const handle = pi.ui.custom(myComponent);
  // ...
  handle.close();
}
```

## Overlays

オーバーレイは画面全体をクリアせず、既存コンテンツの上にコンポーネントを描画します。`ctx.ui.custom()` に `{ overlay: true }` を渡してください。

```typescript
const result = await ctx.ui.custom<string | null>(
  (tui, theme, keybindings, done) => new MyDialog({ onClose: done }),
  { overlay: true }
);
```

位置とサイズには `overlayOptions` を使います。

```typescript
const result = await ctx.ui.custom<string | null>(
  (tui, theme, keybindings, done) => new SidePanel({ onClose: done }),
  {
    overlay: true,
    overlayOptions: {
      // サイズ: 数値またはパーセンテージ文字列
      width: "50%",          // 端末幅の 50%
      minWidth: 40,          // 最低 40 桁
      maxHeight: "80%",      // 最大 端末高の 80%

      // 位置: アンカー基準（既定値: "center"）
      anchor: "right-center", // 9 方向: center, top-left, top-center など
      offsetX: -2,            // アンカーからのオフセット
      offsetY: 0,

      // またはパーセンテージ / 絶対位置指定
      row: "25%",            // 上から 25%
      col: 10,               // 10 列目

      // マージン
      margin: 2,             // 全辺、または { top, right, bottom, left }

      // レスポンシブ: 狭い端末では非表示
      visible: (termWidth, termHeight) => termWidth >= 80,
    },
    // プログラムから表示制御するためのハンドルを取得
    onHandle: (handle) => {
      // handle.setHidden(true/false) - 表示切り替え
      // handle.hide() - 完全に削除
    },
  }
);
```

### Overlay Lifecycle

オーバーレイコンポーネントは閉じられると破棄されます。参照を再利用せず、毎回新しいインスタンスを作ってください。

```typescript
// 悪い例 - 破棄済み参照を保持してしまう
let menu: MenuComponent;
await ctx.ui.custom((_, __, ___, done) => {
  menu = new MenuComponent(done);
  return menu;
}, { overlay: true });
setActiveComponent(menu);  // Disposed

// 良い例 - 再表示時は再度呼び出す
const showMenu = () => ctx.ui.custom((_, __, ___, done) =>
  new MenuComponent(done), { overlay: true });

await showMenu();  // 最初の表示
await showMenu();  // 「戻る」は単にもう一度呼ぶ
```

アンカー、マージン、重なり、レスポンシブ表示、アニメーションまで含む包括的な例は [overlay-qa-tests.ts](../examples/extensions/overlay-qa-tests.ts) を参照してください。

## Built-in Components

`@earendil-works/pi-tui` から import します。

```typescript
import { Text, Box, Container, Spacer, Markdown } from "@earendil-works/pi-tui";
```

### Text

単語折り返し付きの複数行テキストです。

```typescript
const text = new Text(
  "Hello World",    // content
  1,                // paddingX (default: 1)
  1,                // paddingY (default: 1)
  (s) => bgGray(s)  // optional background function
);
text.setText("Updated");
```

### Box

パディングと背景色を持つコンテナです。

```typescript
const box = new Box(
  1,                // paddingX
  1,                // paddingY
  (s) => bgGray(s)  // background function
);
box.addChild(new Text("Content", 0, 0));
box.setBgFn((s) => bgBlue(s));
```

### Container

子コンポーネントを縦方向にグループ化します。

```typescript
const container = new Container();
container.addChild(component1);
container.addChild(component2);
container.removeChild(component1);
```

### Spacer

空の縦スペースです。

```typescript
const spacer = new Spacer(2);  // 空行 2 行
```

### Markdown

シンタックスハイライト付きで Markdown を描画します。

```typescript
const md = new Markdown(
  "# Title\n\nSome **bold** text",
  1,        // paddingX
  1,        // paddingY
  theme     // MarkdownTheme（後述）
);
md.setText("Updated markdown");
```

### Image

対応端末（Kitty、iTerm2、Ghostty、WezTerm）で画像を描画します。

```typescript
const image = new Image(
  base64Data,   // base64 エンコード済み画像
  "image/png",  // MIME type
  theme,        // ImageTheme
  { maxWidthCells: 80, maxHeightCells: 24 }
);
```

## Keyboard Input

キー判定には `matchesKey()` を使ってください。

```typescript
import { matchesKey, Key } from "@earendil-works/pi-tui";

handleInput(data: string) {
  if (matchesKey(data, Key.up)) {
    this.selectedIndex--;
  } else if (matchesKey(data, Key.enter)) {
    this.onSelect?.(this.selectedIndex);
  } else if (matchesKey(data, Key.escape)) {
    this.onCancel?.();
  } else if (matchesKey(data, Key.ctrl("c"))) {
    // Ctrl+C
  }
}
```

**キー識別子**（補完のために `Key.*` を使うか、文字列リテラルを使います）:
- 基本キー: `Key.enter`, `Key.escape`, `Key.tab`, `Key.space`, `Key.backspace`, `Key.delete`, `Key.home`, `Key.end`
- 矢印キー: `Key.up`, `Key.down`, `Key.left`, `Key.right`
- 修飾キー付き: `Key.ctrl("c")`, `Key.shift("tab")`, `Key.alt("left")`, `Key.ctrlShift("p")`
- 文字列形式でも可: `"enter"`, `"ctrl+c"`, `"shift+tab"`, `"ctrl+shift+p"`

## Line Width

**重要:** `render()` が返す各行は `width` パラメータを超えてはいけません。

```typescript
import { visibleWidth, truncateToWidth } from "@earendil-works/pi-tui";

render(width: number): string[] {
  // 長い行を切り詰める
  return [truncateToWidth(this.text, width)];
}
```

ユーティリティ:
- `visibleWidth(str)` - 表示幅を取得する（ANSI コードは無視）
- `truncateToWidth(str, width, ellipsis?)` - 任意の省略記号付きで切り詰める
- `wrapTextWithAnsi(str, width)` - ANSI コードを保持したまま単語折り返しする

## Creating Custom Components

例: インタラクティブなセレクタ

```typescript
import {
  matchesKey, Key,
  truncateToWidth, visibleWidth
} from "@earendil-works/pi-tui";

class MySelector {
  private items: string[];
  private selected = 0;
  private cachedWidth?: number;
  private cachedLines?: string[];

  public onSelect?: (item: string) => void;
  public onCancel?: () => void;

  constructor(items: string[]) {
    this.items = items;
  }

  handleInput(data: string): void {
    if (matchesKey(data, Key.up) && this.selected > 0) {
      this.selected--;
      this.invalidate();
    } else if (matchesKey(data, Key.down) && this.selected < this.items.length - 1) {
      this.selected++;
      this.invalidate();
    } else if (matchesKey(data, Key.enter)) {
      this.onSelect?.(this.items[this.selected]);
    } else if (matchesKey(data, Key.escape)) {
      this.onCancel?.();
    }
  }

  render(width: number): string[] {
    if (this.cachedLines && this.cachedWidth === width) {
      return this.cachedLines;
    }

    this.cachedLines = this.items.map((item, i) => {
      const prefix = i === this.selected ? "> " : "  ";
      return truncateToWidth(prefix + item, width);
    });
    this.cachedWidth = width;
    return this.cachedLines;
  }

  invalidate(): void {
    this.cachedWidth = undefined;
    this.cachedLines = undefined;
  }
}
```

拡張内での使用例:

```typescript
pi.registerCommand("pick", {
  description: "項目を選択する",
  handler: async (args, ctx) => {
    const items = ["Option A", "Option B", "Option C"];
    const selector = new MySelector(items);

    let handle: { close: () => void; requestRender: () => void };

    await new Promise<void>((resolve) => {
      selector.onSelect = (item) => {
        ctx.ui.notify(`選択: ${item}`, "info");
        handle.close();
        resolve();
      };
      selector.onCancel = () => {
        handle.close();
        resolve();
      };
      handle = ctx.ui.custom(selector);
    });
  }
});
```

## Theming

コンポーネントはスタイリング用にテーマオブジェクトを受け取れます。

**`renderCall` / `renderResult` 内**では `theme` パラメータを使います。

```typescript
renderResult(result, options, theme, context) {
  // 前景色には theme.fg() を使う
  return new Text(theme.fg("success", "Done!"), 0, 0);

  // 背景色には theme.bg() を使う
  const styled = theme.bg("toolPendingBg", theme.fg("accent", "text"));
}
```

**前景色**（`theme.fg(color, text)`）:

| Category | Colors |
|----------|--------|
| General | `text`, `accent`, `muted`, `dim` |
| Status | `success`, `error`, `warning` |
| Borders | `border`, `borderAccent`, `borderMuted` |
| Messages | `userMessageText`, `customMessageText`, `customMessageLabel` |
| Tools | `toolTitle`, `toolOutput` |
| Diffs | `toolDiffAdded`, `toolDiffRemoved`, `toolDiffContext` |
| Markdown | `mdHeading`, `mdLink`, `mdLinkUrl`, `mdCode`, `mdCodeBlock`, `mdCodeBlockBorder`, `mdQuote`, `mdQuoteBorder`, `mdHr`, `mdListBullet` |
| Syntax | `syntaxComment`, `syntaxKeyword`, `syntaxFunction`, `syntaxVariable`, `syntaxString`, `syntaxNumber`, `syntaxType`, `syntaxOperator`, `syntaxPunctuation` |
| Thinking | `thinkingOff`, `thinkingMinimal`, `thinkingLow`, `thinkingMedium`, `thinkingHigh`, `thinkingXhigh` |
| Modes | `bashMode` |

**背景色**（`theme.bg(color, text)`）:

`selectedBg`, `userMessageBg`, `customMessageBg`, `toolPendingBg`, `toolSuccessBg`, `toolErrorBg`

**Markdown 用**には `getMarkdownTheme()` を使います。

```typescript
import { getMarkdownTheme } from "@earendil-works/pi-coding-agent";
import { Markdown } from "@earendil-works/pi-tui";

renderResult(result, options, theme, context) {
  const mdTheme = getMarkdownTheme();
  return new Markdown(result.details.markdown, 0, 0, mdTheme);
}
```

**カスタムコンポーネント用**には、自前のテーマインターフェースを定義できます。

```typescript
interface MyTheme {
  selected: (s: string) => string;
  normal: (s: string) => string;
}
```

## Debug logging

stdout に書き込まれる生の ANSI ストリームを記録するには `PI_TUI_WRITE_LOG` を設定します。

```bash
PI_TUI_WRITE_LOG=/tmp/tui-ansi.log npx tsx packages/tui/test/chat-simple.ts
```

## Performance

可能な限り描画結果をキャッシュしてください。

```typescript
class CachedComponent {
  private cachedWidth?: number;
  private cachedLines?: string[];

  render(width: number): string[] {
    if (this.cachedLines && this.cachedWidth === width) {
      return this.cachedLines;
    }
    // ... 行を計算 ...
    this.cachedWidth = width;
    this.cachedLines = lines;
    return lines;
  }

  invalidate(): void {
    this.cachedWidth = undefined;
    this.cachedLines = undefined;
  }
}
```

状態が変化したら `invalidate()` を呼び、その後 `handle.requestRender()` で再描画をトリガーしてください。

## Invalidation and Theme Changes

テーマが変わると、TUI はすべてのコンポーネントに対して `invalidate()` を呼び出し、キャッシュをクリアします。テーマ変更を反映させるには、コンポーネント側で `invalidate()` を正しく実装している必要があります。

### The Problem

コンポーネントが `theme.fg()` や `theme.bg()` などを使ってテーマ色を文字列へ焼き込み、それをキャッシュしている場合、そのキャッシュ済み文字列には古いテーマの ANSI エスケープコードが含まれます。テーマ付きコンテンツを別に保持していると、描画キャッシュを消すだけでは不十分です。

**悪い例**（テーマ色が更新されない）:

```typescript
class BadComponent extends Container {
  private content: Text;

  constructor(message: string, theme: Theme) {
    super();
    // テーマ色を焼き込んだ文字列を Text コンポーネントに保存してしまう
    this.content = new Text(theme.fg("accent", message), 1, 0);
    this.addChild(this.content);
  }
  // invalidate をオーバーライドしていないため、親の invalidate では
  // 子の描画キャッシュしか消えず、焼き込み済みコンテンツは残る
}
```

### The Solution

テーマ色を使ってコンテンツを組み立てるコンポーネントは、`invalidate()` が呼ばれたときにそのコンテンツも再構築しなければなりません。

```typescript
class GoodComponent extends Container {
  private message: string;
  private content: Text;

  constructor(message: string) {
    super();
    this.message = message;
    this.content = new Text("", 1, 0);
    this.addChild(this.content);
    this.updateDisplay();
  }

  private updateDisplay(): void {
    // 現在のテーマでコンテンツを再構築する
    this.content.setText(theme.fg("accent", this.message));
  }

  override invalidate(): void {
    super.invalidate();  // 子のキャッシュをクリア
    this.updateDisplay(); // 新しいテーマで再構築
  }
}
```

### Pattern: Rebuild on Invalidate

複雑なコンテンツを持つコンポーネントでは次のパターンを使います。

```typescript
class ComplexComponent extends Container {
  private data: SomeData;

  constructor(data: SomeData) {
    super();
    this.data = data;
    this.rebuild();
  }

  private rebuild(): void {
    this.clear();  // すべての子を削除

    // 現在のテーマで UI を構築
    this.addChild(new Text(theme.fg("accent", theme.bold("Title")), 1, 0));
    this.addChild(new Spacer(1));

    for (const item of this.data.items) {
      const color = item.active ? "success" : "muted";
      this.addChild(new Text(theme.fg(color, item.label), 1, 0));
    }
  }

  override invalidate(): void {
    super.invalidate();
    this.rebuild();
  }
}
```

### When This Matters

このパターンが必要なのは次のような場合です。

1. **テーマ色の焼き込み** - `theme.fg()` や `theme.bg()` を使って、子コンポーネント内へ保存するスタイル済み文字列を作る場合
2. **シンタックスハイライト** - テーマ依存の構文色を適用する `highlightCode()` を使う場合
3. **複雑なレイアウト** - テーマ色を埋め込んだ子コンポーネントツリーを組み立てる場合

このパターンが不要なのは次のような場合です。

1. **テーマコールバックの利用** - `(text) => theme.fg("accent", text)` のように、描画時に呼ばれる関数を渡すだけの場合
2. **単純なコンテナ** - テーマ付きコンテンツを追加せず、他コンポーネントを束ねるだけの場合
3. **状態を持たない描画** - 毎回の `render()` 呼び出しで新しくテーマ付き出力を計算し、キャッシュしない場合

## Common Patterns

これらのパターンは、拡張でよくある UI 要件をカバーします。**ゼロから作るより、まずこれらをコピーして使ってください。**

### Pattern 1: Selection Dialog (SelectList)

選択肢の一覧からユーザーに選ばせるためのパターンです。`@earendil-works/pi-tui` の `SelectList` を、枠線用の `DynamicBorder` と組み合わせて使います。

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { DynamicBorder } from "@earendil-works/pi-coding-agent";
import { Container, type SelectItem, SelectList, Text } from "@earendil-works/pi-tui";

pi.registerCommand("pick", {
  handler: async (_args, ctx) => {
    const items: SelectItem[] = [
      { value: "opt1", label: "Option 1", description: "First option" },
      { value: "opt2", label: "Option 2", description: "Second option" },
      { value: "opt3", label: "Option 3" },  // description is optional
    ];

    const result = await ctx.ui.custom<string | null>((tui, theme, _kb, done) => {
      const container = new Container();

      // 上側の枠線
      container.addChild(new DynamicBorder((s: string) => theme.fg("accent", s)));

      // タイトル
      container.addChild(new Text(theme.fg("accent", theme.bold("Pick an Option")), 1, 0));

      // テーマ付き SelectList
      const selectList = new SelectList(items, Math.min(items.length, 10), {
        selectedPrefix: (t) => theme.fg("accent", t),
        selectedText: (t) => theme.fg("accent", t),
        description: (t) => theme.fg("muted", t),
        scrollInfo: (t) => theme.fg("dim", t),
        noMatch: (t) => theme.fg("warning", t),
      });
      selectList.onSelect = (item) => done(item.value);
      selectList.onCancel = () => done(null);
      container.addChild(selectList);

      // ヘルプテキスト
      container.addChild(new Text(theme.fg("dim", "↑↓ navigate • enter select • esc cancel"), 1, 0));

      // 下側の枠線
      container.addChild(new DynamicBorder((s: string) => theme.fg("accent", s)));

      return {
        render: (w) => container.render(w),
        invalidate: () => container.invalidate(),
        handleInput: (data) => { selectList.handleInput(data); tui.requestRender(); },
      };
    });

    if (result) {
      ctx.ui.notify(`Selected: ${result}`, "info");
    }
  },
});
```

**例:** [preset.ts](../examples/extensions/preset.ts), [tools.ts](../examples/extensions/tools.ts)

### Pattern 2: Async Operation with Cancel (BorderedLoader)

時間のかかる処理をキャンセル可能にしたいときのパターンです。`BorderedLoader` はスピナーを表示し、escape でのキャンセルも処理します。

```typescript
import { BorderedLoader } from "@earendil-works/pi-coding-agent";

pi.registerCommand("fetch", {
  handler: async (_args, ctx) => {
    const result = await ctx.ui.custom<string | null>((tui, theme, _kb, done) => {
      const loader = new BorderedLoader(tui, theme, "データを取得中...");
      loader.onAbort = () => done(null);

      // 非同期処理を実行
      fetchData(loader.signal)
        .then((data) => done(data))
        .catch(() => done(null));

      return loader;
    });

    if (result === null) {
      ctx.ui.notify("キャンセルされました", "info");
    } else {
      ctx.ui.setEditorText(result);
    }
  },
});
```

**例:** [qna.ts](../examples/extensions/qna.ts), [handoff.ts](../examples/extensions/handoff.ts)

### Pattern 3: Settings/Toggles (SettingsList)

複数の設定を切り替えるためのパターンです。`@earendil-works/pi-tui` の `SettingsList` を `getSettingsListTheme()` と組み合わせて使います。

```typescript
import { getSettingsListTheme } from "@earendil-works/pi-coding-agent";
import { Container, type SettingItem, SettingsList, Text } from "@earendil-works/pi-tui";

pi.registerCommand("settings", {
  handler: async (_args, ctx) => {
    const items: SettingItem[] = [
      { id: "verbose", label: "Verbose mode", currentValue: "off", values: ["on", "off"] },
      { id: "color", label: "Color output", currentValue: "on", values: ["on", "off"] },
    ];

    await ctx.ui.custom((_tui, theme, _kb, done) => {
      const container = new Container();
      container.addChild(new Text(theme.fg("accent", theme.bold("Settings")), 1, 1));

      const settingsList = new SettingsList(
        items,
        Math.min(items.length + 2, 15),
        getSettingsListTheme(),
        (id, newValue) => {
          // 値変更時の処理
          ctx.ui.notify(`${id} = ${newValue}`, "info");
        },
        () => done(undefined),  // 閉じたとき
        { enableSearch: true }, // 任意: ラベルに対するあいまい検索を有効化
      );
      container.addChild(settingsList);

      return {
        render: (w) => container.render(w),
        invalidate: () => container.invalidate(),
        handleInput: (data) => settingsList.handleInput?.(data),
      };
    });
  },
});
```

**例:** [tools.ts](../examples/extensions/tools.ts)

### Pattern 4: Persistent Status Indicator

再描画をまたいで残るステータスをフッターに表示します。モード表示などに向いています。

```typescript
// ステータスを設定（フッターに表示）
ctx.ui.setStatus("my-ext", ctx.ui.theme.fg("accent", "● active"));

// クリア
ctx.ui.setStatus("my-ext", undefined);
```

**例:** [status-line.ts](../examples/extensions/status-line.ts), [plan-mode/](../examples/extensions/plan-mode/), [preset.ts](../examples/extensions/preset.ts)

### Pattern 4b: Working Indicator Customization

pi が応答をストリーミングしている間に表示されるインライン作業インジケータをカスタマイズします。

```typescript
// 固定インジケータ
ctx.ui.setWorkingIndicator({ frames: [ctx.ui.theme.fg("accent", "●")] });

// カスタムアニメーションインジケータ
ctx.ui.setWorkingIndicator({
  frames: [
    ctx.ui.theme.fg("dim", "·"),
    ctx.ui.theme.fg("muted", "•"),
    ctx.ui.theme.fg("accent", "●"),
    ctx.ui.theme.fg("muted", "•"),
  ],
  intervalMs: 120,
});

// インジケータを完全に非表示
ctx.ui.setWorkingIndicator({ frames: [] });

// pi のデフォルトスピナーを復元
ctx.ui.setWorkingIndicator();
```

これは通常のストリーミング時の作業インジケータにのみ影響します。圧縮や再試行のローダーは組み込みスタイルのままです。カスタムフレームはそのまま描画されるため、必要なら拡張側で色付けも行ってください。

**例:** [working-indicator.ts](../examples/extensions/working-indicator.ts)

### Pattern 5: Widgets Above/Below Editor

入力エディタの上または下に永続コンテンツを表示します。TODO リストや進捗表示に向いています。

```typescript
// シンプルな文字列配列（既定ではエディタ上）
ctx.ui.setWidget("my-widget", ["Line 1", "Line 2"]);

// エディタ下に描画
ctx.ui.setWidget("my-widget", ["Line 1", "Line 2"], { placement: "belowEditor" });

// またはテーマ付き
ctx.ui.setWidget("my-widget", (_tui, theme) => {
  const lines = items.map((item, i) =>
    item.done
      ? theme.fg("success", "✓ ") + theme.fg("muted", item.text)
      : theme.fg("dim", "○ ") + item.text
  );
  return {
    render: () => lines,
    invalidate: () => {},
  };
});

// クリア
ctx.ui.setWidget("my-widget", undefined);
```

**例:** [plan-mode/](../examples/extensions/plan-mode/)

### Pattern 6: Custom Footer

フッターを置き換えます。`footerData` からは、拡張から他では取得できないデータにアクセスできます。

```typescript
ctx.ui.setFooter((tui, theme, footerData) => ({
  invalidate() {},
  render(width: number): string[] {
    // footerData.getGitBranch(): string | null
    // footerData.getExtensionStatuses(): ReadonlyMap<string, string>
    return [`${ctx.model?.id} (${footerData.getGitBranch() || "no git"})`];
  },
  dispose: footerData.onBranchChange(() => tui.requestRender()), // リアクティブ
}));

ctx.ui.setFooter(undefined); // デフォルトへ戻す
```

トークン統計は `ctx.sessionManager.getBranch()` と `ctx.model` から取得できます。

**例:** [custom-footer.ts](../examples/extensions/custom-footer.ts)

### Pattern 7: Custom Editor (vim mode, etc.)

入力エディタ自体を置き換えるパターンです。vim ライクな編集など、エディタ体験を丸ごと差し替えたい場合に使います。

```typescript
import { CustomEditor } from "@earendil-works/pi-coding-agent";

class VimEditor extends CustomEditor {
  handleInput(data: string): void {
    if (/* vim キーを処理 */) {
      // カスタム処理
      this.invalidate();
      this.tui.requestRender();
      return;
    }

    // 未処理キーは親へ委譲して組み込みキーバインドを維持
    super.handleInput(data);
  }
}

pi.registerCommand("vim-mode", {
  handler: async (_args, ctx) => {
    // ファクトリはアプリから theme と keybindings を受け取る
    ctx.ui.setEditorComponent((tui, theme, keybindings) =>
      new VimEditor(theme, keybindings)
    );
  });
}
```

**要点:**

- **`CustomEditor` を継承する** - ベースの `Editor` ではなく `CustomEditor` を継承すると、アプリ側キーバインド（escape で中断、ctrl+d で終了、モデル切り替えなど）を利用できます
- **`super.handleInput(data)` を呼ぶ** - 自前で処理しないキーは親へ渡してください
- **ファクトリパターン** - `setEditorComponent` には `tui`、`theme`、`keybindings` を受け取るファクトリ関数を渡します
- **`undefined` を渡して元へ戻す** - `ctx.ui.setEditorComponent(undefined)` で既定エディタを復元します

**例:** [modal-editor.ts](../examples/extensions/modal-editor.ts)

## Key Rules

1. **テーマは常にコールバック引数から使う** - テーマを直接 import せず、`ctx.ui.custom((tui, theme, keybindings, done) => ...)` の `theme` を使ってください。
2. **`DynamicBorder` の色引数は必ず型注釈を付ける** - `(s) => theme.fg("accent", s)` ではなく `(s: string) => theme.fg("accent", s)` と書いてください。
3. **状態変更後は `tui.requestRender()` を呼ぶ** - `handleInput` で状態更新したら、その後に `tui.requestRender()` を呼んでください。
4. **3 メソッドオブジェクトを返す** - カスタムコンポーネントは `{ render, invalidate, handleInput }` を返す必要があります。
5. **既存コンポーネントを使う** - `SelectList`、`SettingsList`、`BorderedLoader` で大半のケースをまかなえます。作り直さないでください。

## Examples

- **選択 UI**: [examples/extensions/preset.ts](../examples/extensions/preset.ts) - `DynamicBorder` 付きの `SelectList`
- **キャンセル可能な非同期処理**: [examples/extensions/qna.ts](../examples/extensions/qna.ts) - LLM 呼び出しに `BorderedLoader`
- **設定トグル**: [examples/extensions/tools.ts](../examples/extensions/tools.ts) - ツール有効/無効切り替えに `SettingsList`
- **ステータス表示**: [examples/extensions/plan-mode/](../examples/extensions/plan-mode/) - `setStatus` と `setWidget`
- **作業インジケータ**: [examples/extensions/working-indicator.ts](../examples/extensions/working-indicator.ts) - `setWorkingIndicator`
- **カスタムフッター**: [examples/extensions/custom-footer.ts](../examples/extensions/custom-footer.ts) - 統計付き `setFooter`
- **カスタムエディタ**: [examples/extensions/modal-editor.ts](../examples/extensions/modal-editor.ts) - Vim ライクなモーダル編集
- **Snake ゲーム**: [examples/extensions/snake.ts](../examples/extensions/snake.ts) - キーボード入力とゲームループを持つ完全なゲーム
- **カスタムツール描画**: [examples/extensions/todo.ts](../examples/extensions/todo.ts) - `renderCall` と `renderResult`
