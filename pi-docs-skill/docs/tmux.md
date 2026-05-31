# tmux 設定

> 要約: Pi は tmux 内でも動作しますが、既定設定では修飾キー情報が欠落しやすく、特に `Shift+Enter` や `Ctrl+Enter` が潰れます。このページは、推奨設定 `extended-keys-format csi-u` とその理由を説明します。

Pi は tmux 内でも動きますが、tmux は既定では一部キーの修飾情報を取り除きます。設定なしでは、`Shift+Enter` や `Ctrl+Enter` は通常の `Enter` と区別できないのが一般的です。

## 推奨設定

`~/.tmux.conf` に次を追加します。

```tmux
set -g extended-keys on
set -g extended-keys-format csi-u
```

その後、tmux を完全に再起動します。

```bash
tmux kill-server
tmux
```

Kitty keyboard protocol が利用できない場合、Pi は自動的に拡張キー報告を要求します。`extended-keys-format csi-u` を使うと、tmux は修飾キーを CSI-u 形式で転送し、これが最も信頼できる設定です。

## `csi-u` を推奨する理由

次だけを設定した場合:

```tmux
set -g extended-keys on
```

tmux の既定は `extended-keys-format xterm` です。アプリケーションが拡張キー報告を要求すると、修飾キーは xterm の `modifyOtherKeys` 形式で転送されます。たとえば次のようになります。

- `Ctrl+C` → `\x1b[27;5;99~`
- `Ctrl+D` → `\x1b[27;5;100~`
- `Ctrl+Enter` → `\x1b[27;5;13~`

`extended-keys-format csi-u` を使うと、同じキーは次のように転送されます。

- `Ctrl+C` → `\x1b[99;5u`
- `Ctrl+D` → `\x1b[100;5u`
- `Ctrl+Enter` → `\x1b[13;5u`

Pi は両形式をサポートしますが、tmux の設定としては `csi-u` を推奨します。

## これで解決すること

tmux の拡張キーが無効だと、修飾付き Enter は従来のシーケンスへ潰れます。

| キー | extkeys なし | `csi-u` あり |
|-----|-----------------|--------------|
| Enter | `\r` | `\r` |
| Shift+Enter | `\r` | `\x1b[13;2u` |
| Ctrl+Enter | `\r` | `\x1b[13;5u` |
| Alt/Option+Enter | `\x1b\r` | `\x1b[13;3u` |

これは既定キーバインド（`Enter` で送信、`Shift+Enter` で改行）や、修飾付き Enter を使うカスタムキーバインドすべてに影響します。

## 要件

- tmux 3.2 以降（`tmux -V` で確認）
- 拡張キーをサポートするターミナルエミュレータ（Ghostty、Kitty、iTerm2、WezTerm、Windows Terminal）
