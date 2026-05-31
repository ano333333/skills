# ターミナル設定

> 要約: Pi は修飾キーを安定して認識するために Kitty keyboard protocol を前提にします。このページでは、主要ターミナルごとの設定方法と、`Shift+Enter` や `Alt+Enter` が効かない場合の対処をまとめています。

Pi は信頼できる修飾キー検出のために [Kitty keyboard protocol](https://sw.kovidgoyal.net/kitty/keyboard-protocol/) を使います。ほとんどのモダンターミナルはこのプロトコルをサポートしていますが、一部では設定が必要です。

## Kitty, iTerm2

そのままで動作します。

## Apple Terminal

Pi は利用可能な場合に拡張キー報告を有効化します。Terminal.app がそれでも `Shift+Enter` を通常の Return として送る場合、Pi はローカル macOS の modifier 情報を使うフォールバックにより、その Return を `Shift+Enter` として扱います。

このフォールバックは、Pi が Terminal.app と同じ Mac 上で動いている場合にのみ有効です。リモート SSH 越しではローカルキーボードを検出できません。

## Ghostty

Ghostty 設定に次を追加します（macOS では `~/Library/Application Support/com.mitchellh.ghostty/config`、Linux では `~/.config/ghostty/config`）。

```text
keybind = alt+backspace=text:\x1b\x7f
```

古い Claude Code では、次の Ghostty マッピングを追加していた可能性があります。

```text
keybind = shift+enter=text:\n
```

このマッピングは生の linefeed バイトを送信します。Pi 内ではそれを `Ctrl+J` と区別できないため、tmux と Pi のどちらからも本来の `shift+enter` キーイベントとして見えなくなります。

Claude Code 2.x 以降のためだけにこのマッピングを入れていたなら、tmux 内で Claude Code を使いたい場合を除き、削除できます。tmux 内ではその Ghostty マッピングが引き続き必要です。

そのリマップ経由で tmux でも `Shift+Enter` を使い続けたい場合は、`~/.pi/agent/keybindings.json` の `newLine` キーバインドへ `ctrl+j` を追加してください。

```json
{
  "newLine": ["shift+enter", "ctrl+j"]
}
```

## WezTerm

`~/.wezterm.lua` を作成します。

```lua
local wezterm = require 'wezterm'
local config = wezterm.config_builder()
config.enable_kitty_keyboard = true
return config
```

WSL 上では、IME 候補ウィンドウの位置決めに表示中のハードウェアカーソルが必要な場合があります。CJK IME の候補がテキストカーソルに追従しない場合は、Pi 起動前に `PI_HARDWARE_CURSOR=1` を設定するか、設定で `showHardwareCursor` を `true` にしてください。

## VS Code (統合ターミナル)

`keybindings.json` の配置場所:

- macOS: `~/Library/Application Support/Code/User/keybindings.json`
- Linux: `~/.config/Code/User/keybindings.json`
- Windows: `%APPDATA%\\Code\\User\\keybindings.json`

複数行入力で `Shift+Enter` を有効にするには、`keybindings.json` に次を追加します。

```json
{
  "key": "shift+enter",
  "command": "workbench.action.terminal.sendSequence",
  "args": { "text": "\u001b[13;2u" },
  "when": "terminalFocus"
}
```

## Windows Terminal

Pi が使う修飾付き Enter キーを転送するため、`settings.json`（Ctrl+Shift+, または Settings → Open JSON file）に次を追加します。

```json
{
  "actions": [
    {
      "command": { "action": "sendInput", "input": "\u001b[13;2u" },
      "keys": "shift+enter"
    },
    {
      "command": { "action": "sendInput", "input": "\u001b[13;3u" },
      "keys": "alt+enter"
    }
  ]
}
```

- `Shift+Enter` は改行を挿入します。
- Windows Terminal では `Alt+Enter` が既定で全画面表示に割り当てられています。そのため Pi は follow-up キュー用の `Alt+Enter` を受け取れません。
- `Alt+Enter` を `sendInput` に再割り当てすると、本来のキー組み合わせが Pi に転送されます。

すでに `actions` 配列がある場合は、その中にオブジェクトを追加してください。古い全画面挙動が残る場合は、Windows Terminal を完全に終了してから再起動します。

## xfce4-terminal, terminator

これらのターミナルはエスケープシーケンスのサポートが限定的です。`Ctrl+Enter` や `Shift+Enter` のような修飾付き Enter を通常の `Enter` と区別できないため、`submit: ["ctrl+enter"]` のようなカスタムキーバインドは機能しません。

最良の体験のためには、Kitty keyboard protocol をサポートするターミナルを使ってください。

- [Kitty](https://sw.kovidgoyal.net/kitty/)
- [Ghostty](https://ghostty.org/)
- [WezTerm](https://wezfurlong.org/wezterm/)
- [iTerm2](https://iterm2.com/)
- [Alacritty](https://github.com/alacritty/alacritty)（Kitty protocol 対応付きでビルドが必要）

## IntelliJ IDEA (統合ターミナル)

組み込みターミナルはエスケープシーケンスのサポートが限定的です。IntelliJ のターミナルでは Shift+Enter を Enter と区別できません。

ハードウェアカーソルを表示したい場合は、Pi 起動前に `PI_HARDWARE_CURSOR=1` を設定してください（既定では互換性のため無効）。

最良の体験のためには、専用のターミナルエミュレータを使うことを検討してください。
