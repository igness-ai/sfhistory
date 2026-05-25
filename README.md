# sfhistory

Salesforce の全開発工程をコンテキスト付きでバージョン管理できるCLI。

## Overview

sfhistory は Salesforce プロジェクトのバージョンとコンテキストを管理するツールです。
git のコミットごとに、Salesforce の設定変更と AI との会話履歴をひとまとまりで保存します。

- `sfhistory sync <sha>` で Salesforce 組織を過去のコミット時点に戻す
- `sfhistory why <sha>` でその変更の意図（AI との会話）を振り返る

## インストール


| Package manager | Command                                |
| --------------- | -------------------------------------- |
| Homebrew        | `brew install igness-ai/tap/sfhistory` |
| npm             | `npm install -g @igness/sfhistory`  |


## クイックスタート

```bash
sfhistory enable --target-org admin@dev-sandbox.com --yes
git commit -am "your message"
sfhistory log
sfhistory sync <sha> --yes                                      # 全変更を一括
sfhistory sync <sha> force-app/main/default/flows --yes         # 段階 sync (依存エラー時)
sfhistory why <sha>
```

## コマンド


|                           |                                       |
| ------------------------- | ------------------------------------- |
| `enable` / `disable`      | hook 設置 / 削除                          |
| `status`                  | 有効化状態と target org                     |
| `log`                     | snapshot 一覧                           |
| `sync <sha> [<paths>...]` | Salesforce 組織を target sha に揃える        |
| `why <subject>`           | commit (SHA) または metadata の詳細 + AI 会話 |
| `timeline <metadata>`     | metadata 変更を時系列で表示                    |
| `search <query>`          | FTS5 全文検索（CJK 対応）                     |
| `update`                  | install 方法に応じて最新版へ更新                |


すべて `--json` 対応。
`search <query>` は `--limit <N>` で表示件数を制限できます（未指定なら全件）。

## アップデート

update 方法は install 方法によって異なります。GitHub Release / installer で直接配置した standalone binary は `sfhistory update` で更新できます。Homebrew / npm / Cargo 経由で install した場合、`sfhistory update --check` が推奨コマンドを表示し、実際の更新は各 package manager に委任します。

```bash
sfhistory update                        # standalone install の最新版へ
SFHISTORY_NO_UPDATE_CHECK=1 sfhistory   # 起動時通知を抑止
```
