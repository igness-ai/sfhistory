# sfhistory-releases

Release artifacts for [igness-ai/sfhistory](https://github.com/igness-ai/sfhistory) — Salesforce Time Machine.

> v0.0.x は **Apple Silicon Mac** と **Windows x64** をサポート。

## インストール

### macOS (Homebrew)

```bash
brew install igness-ai/tap/sfhistory
```

### macOS (curl)

```bash
curl --proto '=https' --tlsv1.2 -LsSf \
  https://github.com/igness-ai/sfhistory-releases/releases/latest/download/sfhistory-installer.sh | sh
```

### Windows (PowerShell)

```powershell
irm https://github.com/igness-ai/sfhistory-releases/releases/latest/download/sfhistory-installer.ps1 | iex
```

## 始め方

```bash
# Salesforce プロジェクトのリポジトリ内で有効化
sfhistory enable --target-org admin@dev-sandbox.com --yes

# 普段通り commit すると、裏で snapshot 蓄積
git commit -am "add validation rule"

# 過去時点に戻す
sfhistory log                       # snapshot 一覧
sfhistory sync <sha> --dry-run      # 計画確認
sfhistory sync <sha> --yes          # SF org を target に揃える

# AI 会話の振り返り
sfhistory why <sha>
sfhistory search "validation rule"
```

## ライセンス

Proprietary。各 release の `LICENSE` ファイル参照。
