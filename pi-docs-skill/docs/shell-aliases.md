# シェルエイリアス

> 要約: Pi は `bash -c` の非対話モードでコマンドを実行するため、通常はシェルエイリアスが展開されません。このページは、設定でエイリアス読み込みを有効にする最小構成を示します。

Pi は bash を非対話モード（`bash -c`）で実行するため、既定ではエイリアスを展開しません。

シェルエイリアスを有効にするには、`~/.pi/agent/settings.json` に次を追加します。

```json
{
  "shellCommandPrefix": "shopt -s expand_aliases\neval \"$(grep '^alias ' ~/.zshrc)\""
}
```

パス（`~/.zshrc`、`~/.bashrc` など）は、自分のシェル設定に合わせて調整してください。
