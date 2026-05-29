# sfhistory アーキテクチャ

## 概要

sfhistory は 3 つの crate からなる Rust workspace。
**Salesforce ドメイン層**（`sfhistory-core`）、**Git 操作層**（`sfhistory-git`）、**プレゼンテーション層**（`sfhistory-cli`）に分離している。

## Crate 依存関係

```
        sfhistory-cli (binary)
         │      │
         ▼      ▼
   sfhistory-git ──→ sfhistory-core
                  │
                  ▼
            (no internal dep)
```

- **sfhistory-core** は外部依存（git, CLI, 将来の MCP）に**依存しない**。pure ドメイン。
- **sfhistory-git** は git 操作のみ。SF 知識を持たない。
- **sfhistory-cli** は両方を統合し、ユーザに見せる層。

## データフロー

### 通常 commit → snapshot 作成

```
ユーザ / AI が git commit
       │
       ▼
.git/hooks/post-commit
       │
       ▼
sfhistory-cli (post-commit subcommand)
       │
       ▼
sfhistory-git::snapshot::create()
       │
       ├── HEAD 情報取得 (git2)
       ├── files in force-app 取得
       └── JSON シリアライズ
       │
       ▼
sfhistory-git::snapshot::commit_to_branch("sfhistory")
       │
       ▼
作業 branch は無傷、別 branch に snapshot 追加
```

### git の動詞 → SF org への反映

sfhistory は git の動詞 (`revert` / `reset` / `restore`) と 1:1 で対応する 3 つの sync 系
サブコマンドを提供する。各 verb は **「どの tree に揃えるか」** を決めるだけで、
SF への deploy 処理は共通コア (`sfhistory-cli/src/commands/sf_sync.rs`) に集約されている。

```
ユーザ / AI が sfhistory revert <sha>  /  sfhistory reset <sha>  /  sfhistory restore <sha> <path>
       │
       ▼
sfhistory-cli の各 verb (commands/{revert,reset,restore}.rs)
       │
       ├── revert : diff_commit_with_parent_at(<sha>)         (git revert と同じ範囲)
       ├── reset  : diff_head_to_target_at(<sha>)             (HEAD..<sha> 全体)
       └── restore: diff_head_to_target_filtered_at(<sha>, paths) (path 単位)
       │
       ▼
sfhistory-core::rollback::build_rollback_plan(diff)
       │
       ├── Added / Modified → deploy_entries  (package.xml)
       ├── Deleted          → destructive_entries (destructiveChanges.xml)
       └── 削除危険 metadata は deletion_warnings に
       │
       ▼
sf_sync::sync_to_sf(plan)         (3 verb 共通)
       │
       ├── plan を JSON / human で表示
       ├── ユーザに確認 (dialoguer Confirm) ※ --yes でスキップ
       ├── package.xml / destructiveChanges.xml 書き出し
       ├── packageDirectories ensure
       ├── sf project deploy start --dry-run                  (validate)
       └── sf project deploy start                            (apply)
       │
       ▼
結果を stdout（人間用 text または `--json`）で通知

# 補足: post-commit hook は SF deploy を一切起こさない（snapshot 作成のみ）。
#       SF への反映はユーザが上記 verb を **明示的に実行した時だけ** 走る。
```

### AI agent からの呼び出し（CLI + JSON）

v0.1 では MCP server は組まない。AI agent（Claude Code / Cursor / Codex / Blaze）は
shell の Bash tool で `sfhistory <cmd> --json` を直接呼ぶ:

```
AI Agent (Claude Code 等)
       │
       │ shell exec: `sfhistory status --json`
       ▼
sfhistory-cli (CLI subcommand)
       │
       ▼
内部で sfhistory-git / sfhistory-core を呼ぶ
       │
       ▼
stdout に JSON を返す（exit code で状態を伝える）
```

利点:
- MCP-aware AI に限定されない（OpenAI Function Calling や自作 script でも使える）
- `assert_cmd` で統合テストできる（MCP server を立てる必要なし）
- v0.2 で MCP server を追加する場合、内部で CLI を呼ぶ薄い wrapper にすれば済む

## ファイルシステム

### ローカル（リポジトリ）

```
my-project/
├── .git/
│   ├── hooks/
│   │   └── post-commit         # sfhistory が install
│   └── refs/
│       └── heads/
│           └── sfhistory   # snapshot 専用 branch
├── .sfhistory/
│   └── config.toml             # sfhistory 設定（target org 等）
├── force-app/                  # ユーザの SF メタデータ
└── ...
```

### グローバル

```
~/.sfhistory/                        # ユーザレベル設定（あれば）
```

> v0.2 で MCP server を追加する際は Claude Code（`~/Library/Application Support/Claude/`）/
> Cursor（`~/.cursor/mcp.json`）等への登録パスを追記する。

## Snapshot 専用 branch の構造

`sfhistory` branch は通常の git branch だが、**作業 branch とは独立**。
1 commit = 1 snapshot。commit message に `[sfhistory] checkpoint <id>` を含める。
ファイルは JSON で各 commit 時の状態を記録。

```
sfhistory の HEAD
├── snapshots/
│   ├── 2026-04-28-a3b2c4d5.json
│   ├── 2026-04-28-b3c1f0e2.json
│   └── ...
└── meta.toml                    # branch メタ情報
```

## エラーハンドリング戦略

各 crate は独自の `Error` 型を持つ:

- `sfhistory_core::Error`：SF ドメインエラー（deploy 失敗、依存性違反等）
- `sfhistory_git::Error`：git 操作エラー
- `sfhistory_cli` 統合層は `anyhow::Result` で受けて user-friendly に変換

panic / unwrap は禁止（`#![deny(clippy::unwrap_used)]`）。
すべてのエラーは `Result` で伝播し、最終的に CLI で人間にわかる文言になる。

## セキュリティ境界

- 入力検証は **境界**で行う（CLI args、config file パース、`sf` CLI 出力）
- 内部関数は信頼できる入力前提でシンプルに保つ
- secrets は OS keyring（`keyring` crate、将来）に保存。設定ファイルには書かない
- Sentry に送るエラーから PII を除去（commit message / file path のみ）

## ロギング戦略

`tracing` + `tracing-subscriber` で structured logging:

- DEBUG: 開発時のみ
- INFO: ユーザに見せる進捗
- WARN: 想定内の異常（Flow active version 等）
- ERROR: revert / reset / restore deploy 失敗 等

`RUST_LOG=sfhistory=debug` で verbose mode。
