# Windows 設定

> 要約: Windows 上の Pi は bash シェルを必要とします。このページは探索順序と、Git Bash 以外を使う場合の `shellPath` 設定方法を示します。

Pi は Windows 上で bash シェルを必要とします。確認される場所（優先順）は次のとおりです。

1. `~/.pi/agent/settings.json` にあるカスタムパス
2. Git Bash (`C:\Program Files\Git\bin\bash.exe`)
3. PATH 上の `bash.exe`（Cygwin、MSYS2、WSL）

多くのユーザーにとっては [Git for Windows](https://git-scm.com/download/win) で十分です。

## カスタムシェルパス

```json
{
  "shellPath": "C:\\cygwin64\\bin\\bash.exe"
}
```
