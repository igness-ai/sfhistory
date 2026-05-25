## [0.1.2] - 2026-05-25

### 変更

- README に段階 sync の使用例を追加しました。
- `sfhistory sync --help` の `paths` 引数の説明を追記しました。

### 追加

- `docs/sf-metadata-rollback.md` (Salesforce メタデータ Rollback ガイド) を作成しました。
- `docs/sf-metadata-rollback.en.md` (英語版) を作成しました。

### 削除

- 開発中の検討メモ (`docs/agent-adapters.md` / `docs/implementation-plan.md` / `docs/mvp.md` / `docs/v0.3-roadmap.md`) を削除しました。

## [0.0.4] - 2026-05-16

### 新機能

- `sfhistory update` を追加。`self-update` は alias として後方互換維持。
  Standalone install (curl install / GitHub Release 直接) は自前で GitHub
  Releases から binary を取得して更新する。Homebrew / npm / Cargo install
  では適切な update command を案内して exit 1 で終了する。
- `--check` で実行のみ確認、`--yes` で対話確認 skip (Standalone のみ)。

### 変更

- 起動時の new-version 通知に、install 方法に応じた update command を
  表示するよう変更。
- release artifact (`tar.gz` / `zip`) の中身を flatten し、`sfhistory` binary
  が archive 直下に入る構造へ変更。Homebrew Formula、npm postinstall、
  および shell / PowerShell installer もこの構造に合わせて更新。

### 修正

- v0.0.3 に含まれる予定だった上記の改修が反映されていなかった問題を修正し、
  v0.0.4 として再 release。

## [0.0.3] - 2026-05-15

v0.0.4 で再 release されたため、本 version の機能改修は v0.0.4 を参照のこと。

## [0.0.2] - 2026-05-14

### 高速化

- `sfhistory enable` の対話モード時、組織一覧表示を高速化。`sf org list` の呼び出しを廃し、
  Salesforce CLI のローカル auth file (`~/.sfdx/<username>.json` と `~/.sfdx/alias.json`) を
  直接読む実装に切り替えた。Node.js cold start を経由しないため、対話プロンプトまで
  体感即時に到達する。ローカル auth が見つからない場合は従来通り `sf org list` に
  フォールバックする。`orgId` の形式チェックで非 auth 系の `.json` を除外する。
- post-commit hook: snapshot index (SQLite) を WAL モードで開くように変更
  (`journal_mode=WAL`, `synchronous=NORMAL`, `temp_store=MEMORY`)。あわせて `upsert` を
  明示 transaction でラップ。metadata の多い commit でも書き込み I/O が削減される。
- `sfhistory sync` の deploy 進捗確認 polling を、固定 2 秒の繰り返しから時間予算ベースの
  指数バックオフに変更 (初回 2 秒、上限 5 秒、合計 180 秒予算)。残り予算で sleep を clamp し、
  timeout エラー文言には経過秒数と target org を含めるようにした。
- release profile の `opt-level` を `"z"` から `"s"` に変更し、起動時間と binary サイズの
  バランスを調整。

### 新機能

- `sfhistory search` に `--limit <N>` オプションを追加。指定時は検索結果を先頭 N 件に
  制限する (未指定なら全件返却、後方互換維持)。

### その他

- 未使用の直接 `tokio` 依存を workspace から削除
  (`self_update` 経由の transitive 依存は維持)。

## [0.0.1] - 2026-05-10

**Initial public preview release.**

### 主な機能
- git のコミットごとに Salesforce 設定変更を `refs/heads/sfhistory` に snapshot 蓄積
- `sfhistory sync <sha>` で Salesforce 組織を過去のコミット時点に戻す
- AI 会話履歴 (Claude Code / Blaze) を snapshot に紐付け → `why <sha>` で振り返り
- FTS5 全文検索 (CJK 対応): `search <query>`
- 起動時の new-version 通知 + `self-update` コマンド
- すべて `--json` 対応 (AI agent 用)

### サポート対象
- macOS (Apple Silicon)
- Windows (x64)

### 配布チャネル
- Homebrew tap: `brew install igness-ai/tap/sfhistory`
- npm: `npm install -g @igness/sfhistory`

### License
- proprietary EULA (Copyright Igness, Inc.)。詳細は [LICENSE](./LICENSE)。
