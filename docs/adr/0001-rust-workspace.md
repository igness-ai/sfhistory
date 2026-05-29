# ADR-0001: Rust workspace と 3-crate 分割

- 状態: 承認
- 日付: 2026-04-28

## 背景

sfhistory は Salesforce 専用の Time Machine。Rust で書く方針は確定。
複数の関心事（git 操作、SF ドメイン、CLI/MCP）が混在するため、
構造を最初に決める必要がある。

## 決定

3 crate の workspace に分割する:

| Crate | 役割 |
|-------|------|
| `sfhistory-core` | SF ドメイン純粋ロジック（メタデータ分類、manifest、依存性、deploy 呼び出し） |
| `sfhistory-git` | git 操作（snapshot 別 branch、hook、revert 検知、diff 計算） |
| `sfhistory-cli` | バイナリ（CLI dispatcher + MCP server） |

### 依存方向

```
sfhistory-cli → sfhistory-git → sfhistory-core
            └─────────→ sfhistory-core
```

`sfhistory-core` は外部 crate（git2, MCP, CLI）に依存しない pure ドメイン層。

## 選択肢と比較

### Option A: 単一 crate（rejected）

- メリット: シンプル、依存管理楽
- デメリット: ドメインと git/CLI が混ざる、テストが書きにくい、再利用困難

### Option B: 2 crate（rejected）

- `sfhistory-lib` + `sfhistory-cli`
- メリット: ある程度の分離
- デメリット: SF ドメインと git 操作が `sfhistory-lib` に同居、責務が混ざる

### Option C: 3 crate（採用）

- メリット: 責務の明確な分離、テスト容易、将来 plugin 化や SDK 化が容易
- デメリット: 初期セットアップ若干複雑、crate 間の境界設計が必要

### Option D: 4+ crate（rejected）

- mcp / cli / git / core / config 等に細分化
- メリット: 究極の分離
- デメリット: MVP 段階では過剰、ボイラープレート増、学習コスト高

## 結論

3 crate に分割し、依存方向を厳格に守ることで:

1. SF ドメインロジックは git や CLI なしで単体テスト可能
2. 将来 sfhistory-core を別バイナリ（例: GUI 版、SaaS 版）に再利用しやすい
3. AI agent（Claude Code 等）が個別 crate を理解しやすい

## 影響

- `Cargo.toml` workspace に 3 つの member 定義
- 各 crate に個別の `Cargo.toml`（lints, dependencies は workspace から継承）
- 内部 dep は `workspace.dependencies.sfhistory-*` で path 指定

## 補足

将来 `sfhistory-mcp` や `sfhistory-data`（v0.2 の record snapshot）等を分離する場合、
このワークスペース内に member を追加する形を取る。
