# Context System 設計

sfhistory の差別化の核となる「**AI agent の会話を git に永続化して、後で取り出せる**」仕組みの設計。

## 設計原則

1. **sfhistory は LLM API を一切呼ばない** — 全処理は決定論的、ローカル
2. **同じデータの 2 表現** — 人間用の表 / LLM 用の `--json`、情報量は同一
3. **agent 側に追加処理を強制しない** — agent が既に保存している transcript を読むだけ
4. **git ネイティブ** — orphan branch + commit trailer で永続化、外部 DB 不要

## 全体フロー

```
┌─────────────────────┐
│ AI Agent            │
│ (Claude Code 等)    │  ← user が普段通り使う、sfhistory 関与なし
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Local transcript    │
│ (~/.claude/...)     │  ← agent が自動で書く（既存挙動）
└─────────────────────┘
           
[user が git commit]
           │
           ▼
┌─────────────────────┐
│ post-commit hook    │
│ → sfhistory capture       │
└──────────┬──────────┘
           │
   ┌───────┴────────┬────────────────────┐
   ▼                ▼                    ▼
┌───────┐    ┌───────────────┐   ┌──────────────┐
│ commit │    │ sfhistory-context  │   │ .sfhistory/       │
│ trailer│    │ orphan branch │   │ index.db     │
│ 追加    │    │ (JSON snapshot)│   │ (SQLite+FTS5)│
└───────┘    └───────────────┘   └──────────────┘
```

## ストレージ階層（3 層）

### Layer 1: git commit trailer（軽い、git ネイティブ）

user の通常 commit message に追記:

```
Add CFO approval to ContactApprovalFlow

Sfhistory-Checkpoint: 2026-04-20-a3f7b2c
Sfhistory-Agent: claude-code
Sfhistory-Session: abc-123
Sfhistory-Attribution: agent=92,human=8
Sfhistory-Metadata: Flow:ContactApproval,ApprovalProcess:ContactApproval
```

特徴:
- git 標準の commit trailer 形式（`Co-Authored-By:` と同型）
- `git log --grep="Sfhistory-Metadata: Flow:ContactApproval"` で即 filter 可能
- DB ゼロでも基本的な検索が可能
- sfhistory が無効化されても commit message は残る（履歴の最低保証）

### Layer 2: `sfhistory-context` orphan branch（永続データ）

orphan branch なので user の working tree に一切影響しない:

```
sfhistory-context branch の HEAD
├── snapshots/
│   ├── 2026-04-20-a3f7b2c.json
│   ├── 2026-04-21-b4d8e2c.json
│   └── ...
└── meta.toml
```

`snapshots/<date>-<sha>.json` の構造:

```json
{
  "schema_version": 2,
  "git_sha": "a3f7b2c...",
  "captured_at": "2026-04-20T10:23:11Z",
  "agent": {
    "name": "claude-code",
    "version": "2.0.5",
    "session_id": "abc-123"
  },
  "user": {
    "name": "takuma-ogura",
    "email": "..."
  },
  "interaction": {
    "prompts": [
      {"role": "user", "content": "Account の Email__c を追加して", "ts": "..."},
      {"role": "assistant", "content": "...", "ts": "..."}
    ],
    "tool_calls": [
      {"name": "Write", "input": {"path": "...Email__c.field-meta.xml"}, "result": "ok"}
    ],
    "reasoning": "..."  // assistant の thinking ブロックがあれば
  },
  "git": {
    "commit_message": "Add Email field to Account",
    "files_changed": ["force-app/.../Email__c.field-meta.xml"],
    "lines_added": 12,
    "lines_removed": 0
  },
  "salesforce": {
    "metadata_changes": [
      {"type": "CustomField", "full_name": "Account.Email__c", "change": "added"}
    ],
    "target_org": "admin@acme.dev"
  },
  "attribution": {
    "agent_lines": 11,
    "human_lines": 1,
    "agent_pct": 92
  }
}
```

### Layer 3: `.sfhistory/index.db`（検索用 SQLite）

ローカルのみ、`.gitignore` 対象。再構築可能（snapshots branch から再生成）。

```sql
CREATE TABLE sessions (
    sha TEXT PRIMARY KEY,
    captured_at INTEGER,
    agent_name TEXT,
    session_id TEXT,
    summary TEXT,           -- 短縮版（200 token 程度）
    full_path TEXT          -- snapshots/<date>-<sha>.json への相対 path
);

CREATE TABLE metadata_changes (
    sha TEXT,
    metadata_type TEXT,    -- "CustomField" 等
    full_name TEXT,        -- "Account.Email__c"
    change_type TEXT,      -- "added" / "modified" / "deleted"
    FOREIGN KEY (sha) REFERENCES sessions(sha)
);

CREATE TABLE dependencies (
    from_metadata TEXT,    -- "Flow:ContactApproval"
    to_metadata TEXT,      -- "ValidationRule:Foo"
    edge_type TEXT,        -- "references" / "extends" 等
    discovered_at_sha TEXT
);

-- FTS5 全文検索 (SQLite 標準)
CREATE VIRTUAL TABLE fts USING fts5(
    sha UNINDEXED,
    prompt_text,
    reasoning_text,
    commit_message,
    metadata_names
);
```

ベクトル検索は MVP では含めない（FTS5 で十分か検証してから追加）。

## 捕捉フロー（4 hook 体制 + WAL）

「commit しない作業」「`/clear` / `/compact` / ターミナル閉じ / 異常終了」での **データロスを絶対に防ぐ**ため、agent 側 4 hook と WAL (Write-Ahead Log) を併用する。

### 4 hook の役割

| Hook | 発火 | sfhistory record-event の挙動 |
|---|---|---|
| `Stop` | ターン終了ごと | WAL に turn 行を append + fsync |
| `SessionEnd` | `/clear` / `/quit` / ターミナル閉じ 等（matcher 区別なし） | 未 flush WAL を flush → floating snapshot |
| `PreCompact` | `/compact` 手動 / auto-compact 直前（matcher 区別なし） | 同上 |
| `SessionStart` | agent 起動時 | `.sfhistory/wal/` を scan → unflushed あれば自動 flush |

### WAL（Write-Ahead Log）

- 場所: `.sfhistory/wal/<YYYY-MM-DD>/<agent>__<session_id>.jsonl`
- 形式: jsonl, append-only, 1 行 1 event
- `.gitignore` 対象（`.sfhistory/` 配下）
- 書き込み: `O_APPEND` + `fsync` で turn 毎に永続化（ユーザの「データロス絶対なし」要件のため）

WAL の 1 行 schema:

```jsonl
{"ts":"2026-05-17T10:23:11Z","kind":"turn","agent":"claude-code","session_id":"abc-123","events":[...]}
{"ts":"2026-05-17T10:25:42Z","kind":"turn",...}
{"ts":"2026-05-17T10:30:01Z","kind":"flushed","snapshot_kind":"attached","snapshot_sha":"a3f7b2c"}
```

- `events` は [`Event`] (UserPrompt / AssistantText / ToolUse) の配列。snapshot.rs の Event 型を再利用
- `flushed` 行以降のみが「次回 flush 対象」（二重 flush 防止）

### git commit 時のフロー（commit-attached snapshot）

```
git commit (user 操作)
   ↓
.git/hooks/post-commit → sfhistory post-commit
   │
   ├─ 1. WAL を読む（material 優先）
   │   └─ 不在時は active-sessions.json + adapter で fallback
   │
   ├─ 2. snapshot を組成（snapshot_kind: Attached）
   │
   ├─ 3. 4 層に保存
   │   ├─ commit trailer 追加
   │   ├─ sfhistory orphan branch に commit (snapshots/by-sha/<date>-<sha>.json)
   │   ├─ SQLite + FTS5 index
   │   └─ 同範囲を覆う既存 floating snapshot に `superseded_by: <sha>` を書き戻し
   │
   ├─ 4. WAL に `kind: flushed` 行を追記
   │
   └─ 5. 完了報告
```

### SessionEnd / PreCompact 時のフロー（floating snapshot）

```
agent 側 hook 発火
   ↓
sfhistory record-event --agent <kind>
   │
   ├─ 1. WAL から未 flush events を読む
   ├─ 2. snapshot を組成（snapshot_kind: Floating, git_sha: null）
   ├─ 3. orphan branch に commit (snapshots/floating/<date>-<session>.json)
   ├─ 4. SQLite に登録（superseded_by null のみが why/timeline 表示対象）
   └─ 5. WAL に `kind: flushed` 行を追記
```

### SessionStart 時のフロー（起動時リカバリ）

```
agent 起動
   ↓
sfhistory record-event --agent <kind> (hook_event_name: SessionStart)
   │
   ├─ 1. `.sfhistory/wal/` を scan
   ├─ 2. flushed 行で終わっていない jsonl を発見
   └─ 3. 各 unflushed WAL について floating snapshot を作成
```

これで `kill -9` / OS crash / バッテリー切れでも、**次回起動時に必ず救済**される。

実装: Rust の `git2` crate で TreeBuilder → commit_create → reference_set。HEAD 切り替え不要。

## 取り出しフロー（why / timeline / search）

### `sfhistory why <metadata>`

```
1. SQLite で metadata_changes を絞り込み:
   SELECT sha FROM metadata_changes WHERE full_name = 'Flow:ContactApproval'
        OR full_name LIKE 'Flow:Contact%'  -- ワイルドカード許容

2. 各 sha について:
   ├─ sessions テーブルから summary 取得
   └─ snapshots/<date>-<sha>.json から full data 取得

3. 時系列で並べる（captured_at 昇順）

4. dependencies テーブルから関連 metadata を後ろに付ける

5. 出力: human フォーマット or --json
```

### `sfhistory timeline <metadata>`

`why` とほぼ同じだが、時系列をビジュアル中心に表示。

### `sfhistory search <keyword>`

`--limit <N>` 指定時のみ件数を制限する。未指定時は後方互換のため全件を返す。

```
1. FTS5 で全文検索:
   SELECT sha FROM fts WHERE fts MATCH '"承認" OR "approval"' ORDER BY rank [LIMIT N]

2. 各結果について snapshot を読み出し
3. 出力
```

将来的にベクトル検索を加える場合:
- `sqlite-vec` extension で embedding を vector index に追加
- `embed-cmd` (TBD: ローカル fastembed-rs or cloud embedding) で query を embedding 化
- hybrid (BM25 + vector) で ranking

## 出力フォーマット例

### `sfhistory why Flow:ContactApproval`（human）

```
Flow:ContactApproval の context (3 sessions)

[2026-01-15] 初版作成 (a3f7b2c)
  Agent: claude-code (session abc-123)
  Prompt: "Contact ベースの承認フローを作って..."
  変更: +127 行 / -0 行
  関連: ApprovalProcess:ContactApproval, ValidationRule:Foo

[2026-04-20] 5 万円ルール追加 (b3c1f0e)
  Agent: claude-code (session def-456)
  Prompt: "X 社要件で 5 万円以上は CFO 承認..."
  Reasoning: "既存ルール変更でなく新規追加で衝突回避"
  変更: +43 行 / -3 行

[2026-07-03] 通知メール改善 (c8d2e1f)
  Agent: human (no AI context)
  Message: "Improve notification email"
  変更: +15 行 / -8 行

依存:
  Flow:ContactApproval
    └─→ ApprovalProcess:ContactApproval
          └─→ ValidationRule:Foo
                └─→ CustomField:Account.ApprovalThreshold__c

引用: a3f7b2c, b3c1f0e, c8d2e1f
```

### `sfhistory why Flow:ContactApproval --json`

```json
{
  "metadata": "Flow:ContactApproval",
  "history": [ /* 3 sessions の構造化データ */ ],
  "dependencies": { /* グラフ */ },
  "citations": ["a3f7b2c", "b3c1f0e", "c8d2e1f"]
}
```

LLM agent はこの JSON を読んで自然言語で再構成する想定。

## エラー / 失敗ケース

| ケース | 挙動 |
|---|---|
| transcript file が見つからない | warn 表示、human commit として記録 |
| transcript parse 失敗 | error 表示、commit message のみ保存して継続 |
| sfhistory-context branch への書き込み失敗 | error → snapshot は SQLite にのみ残し、後で再 push 可 |
| index.db 破損 | snapshots branch から `sfhistory reindex` で再構築 |
| post-commit hook 自体が失敗 | git commit は成功する（hook の失敗は commit を止めない） |

## 安全性

- **HEAD / working tree は一切触らない**（git plumbing 経由）
- **secrets を捕捉しない**（transcript に含まれていれば写ってしまうので、user 側で agent の secrets handling に注意）
- **`.sfhistory/` は `.gitignore` に追加**（index.db は再構築可能なので共有不要）
- **sfhistory-context branch を push するか否かは user 任意**

## push 戦略（チーム共有）

```
個人作業:
  sfhistory-context は local のみ。push しない。

チーム共有:
  git push origin sfhistory-context
  GitHub UI で見ても羅列されるだけだが、sfhistory CLI で query すれば実用的
  チームメンバーが pull すれば全員の context を共有
```

cloud sync は MVP に含めない（user の方針: 完全無料、運営コスト 0）。

## 関連 doc

- [mvp.md](./mvp.md) — MVP の全体像
- [agent-adapters.md](./agent-adapters.md) — 各 agent の transcript 形式と adapter 実装
- [json-schema.md](./json-schema.md) — `--json` 出力スキーマ
