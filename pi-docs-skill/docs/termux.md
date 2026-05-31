# Termux (Android) 設定

> 要約: Pi は Android 上でも Termux を通じて動作します。このページは、必要アプリ、導入手順、クリップボード制約、`AGENTS.md` の例、ストレージ権限や Node.js 周りのトラブルシュートをまとめています。

Pi は Android 上で [Termux](https://termux.dev/) を使って動作します。Termux は Android 向けのターミナルエミュレータ兼 Linux 環境です。

## 前提条件

1. [Termux](https://github.com/termux/termux-app#installation) を GitHub または F-Droid からインストールする（Google Play 版は非推奨）
2. クリップボードなどの端末連携のために [Termux:API](https://github.com/termux/termux-api#installation) を GitHub または F-Droid からインストールする

## インストール

```bash
# パッケージを更新
pkg update && pkg upgrade

# 依存関係をインストール
pkg install nodejs termux-api git

# pi をインストール
npm install -g --ignore-scripts @earendil-works/pi-coding-agent

# 設定ディレクトリを作成
mkdir -p ~/.pi/agent

# pi を実行
pi
```

## クリップボード対応

Termux 上で動いている場合、クリップボード操作には `termux-clipboard-set` と `termux-clipboard-get` を使います。これらを機能させるには、Termux:API アプリがインストールされている必要があります。

Termux では画像クリップボードはサポートされていません（`ctrl+v` による画像貼り付け機能は使えません）。

## Termux 向け AGENTS.md の例

Termux 環境をエージェントに理解させるため、`~/.pi/agent/AGENTS.md` を作成します。

````markdown
# Agent Environment: Termux on Android

## Location
- **OS**: Android (Termux terminal emulator)
- **Home**: `/data/data/com.termux/files/home`
- **Prefix**: `/data/data/com.termux/files/usr`
- **Shared storage**: `/storage/emulated/0` (Downloads, Documents, etc.)

## Opening URLs
```bash
termux-open-url "https://example.com"
```

## Opening Files
```bash
termux-open file.pdf          # Opens with default app
termux-open --chooser image.jpg      # Choose app
```

## Clipboard
```bash
termux-clipboard-set "text"   # Copy
termux-clipboard-get          # Paste
```

## Notifications
```bash
termux-notification -t "Title" -c "Content"
```

## Device Info
```bash
termux-battery-status         # Battery info
termux-wifi-connectioninfo    # WiFi info
termux-telephony-deviceinfo   # Device info
```

## Sharing
```bash
termux-share -a send file.txt # Share file
```

## Other Useful Commands
```bash
termux-toast "message"        # Quick toast popup
termux-vibrate                # Vibrate device
termux-tts-speak "hello"      # Text to speech
termux-camera-photo out.jpg   # Take photo
```

## Notes
- Termux:API app must be installed for `termux-*` commands
- Use `pkg install termux-api` for the command-line tools
- Storage permission needed for `/storage/emulated/0` access
````

## 制限事項

- **画像クリップボードなし**: Termux のクリップボード API はテキストのみ対応
- **ネイティブバイナリなし**: クリップボードモジュールのような一部のオプション依存ネイティブバイナリは Android ARM64 では利用できず、インストール時にスキップされる
- **ストレージアクセス**: `/storage/emulated/0`（Downloads など）にアクセスするには、一度 `termux-setup-storage` を実行して権限を付与する必要がある

## トラブルシューティング

### クリップボードが動かない

次の 2 つのアプリがインストールされていることを確認してください。

1. Termux（GitHub または F-Droid 版）
2. Termux:API（GitHub または F-Droid 版）

その後、CLI ツールをインストールします。

```bash
pkg install termux-api
```

### 共有ストレージで Permission denied が出る

一度だけ次を実行して、ストレージ権限を付与します。

```bash
termux-setup-storage
```

### Node.js のインストールに失敗する

npm が失敗する場合は、キャッシュのクリアを試してください。

```bash
npm cache clean --force
```
