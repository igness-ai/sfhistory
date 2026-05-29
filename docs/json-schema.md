# sfhistory `--json` 出力スキーマ

このドキュメントは AI agent / 自動化スクリプトが sfhistory を呼び出す際の
`--json` 出力スキーマを定義する。**v1.0 までは minor バージョンで breaking 可能**。

## `sfhistory status --json`

```json
{
  "target_org": "admin@acme.dev",
  "config_path": "/path/to/repo/.sfhistory/config.toml",
  "hook": "installed"
}
```

| field | type | description |
|-------|------|-------------|
| `target_org` | string \| null | enable 時に指定された Salesforce org |
| `config_path` | string | 探索した config ファイルの絶対パス |
| `hook` | enum | `"installed"` / `"not_installed"` / `"foreign_hook"` |

**exit code**:
- `0`: enabled（target_org あり + hook=installed）
- `1`: not enabled

## `sfhistory log --json`

```json
[
  {
    "schema_version": 1,
    "created_at": "2026-04-29T04:42:28.411566Z",
    "git_sha": "b487337579c6d37486ec6ddb544598a4092b23fc",
    "git_action": "commit",
    "branch": "feature/account",
    "files": ["force-app/main/default/classes/Foo.cls"],
    "target_org": "admin@acme.dev",
    "label": null
  }
]
```

newest-first の `Snapshot` 配列。`--limit N` で上限指定（default 20）。

## `sfhistory show <sha> --json`

単一 `Snapshot` オブジェクト（`sfhistory log` 配列の 1 要素と同形式）。
SHA prefix で曖昧マッチ。一致なしは exit 1 + stderr に "snapshot not found"。

## `sfhistory revert <sha> --json` / `sfhistory reset <sha> --json` / `sfhistory restore <sha> <path>... --json`

3 つの sync 系 verb は **同一の plan JSON** を出力する（内部の deploy フローを共通化しているため）。
それぞれの違いは「どの diff を計算するか」のみ:

| コマンド | 計算対象 |
|---------|---------|
| `sfhistory revert <sha>` | 指定 commit と親 commit の diff（= 旧 `sfhistory rollback <sha>`） |
| `sfhistory reset <sha>` | HEAD と `<sha>` の tree diff |
| `sfhistory restore <sha> <path>...` | HEAD と `<sha>` の tree diff を `<path>` で絞り込んだもの |

Plan のみ stdout に JSON で出力（実 deploy の進捗は stderr に）:

```json
{
  "deploy_entries": [
    { "metadata_type": "ApexClass", "full_name": "Foo" }
  ],
  "destructive_entries": [
    { "metadata_type": "CustomField", "full_name": "Account.Email__c" }
  ],
  "unclassified_files": [],
  "deletion_warnings": [
    {
      "entry": { "metadata_type": "Flow", "full_name": "MyFlow" },
      "reason": "flow_active"
    }
  ]
}
```

| field | type | description |
|-------|------|-------------|
| `deploy_entries` | `ManifestEntry[]` | `package.xml` に積まれる add / modify |
| `destructive_entries` | `ManifestEntry[]` | `destructiveChanges.xml` に積まれる削除 |
| `unclassified_files` | `string[]` | SF 以外のファイル（無視される） |
| `deletion_warnings` | `DeletionWarning[]` | 削除危険メタデータの警告 |

`reason` は `"flow_active"` / `"access_control"` / `"apex_trigger"` のいずれか。

**exit code**:
- `0`: validate 成功（`--dry-run` 時）または apply 成功
- `1`: validate / apply 失敗、または `--yes` 未指定

## `Snapshot` 型（共通）

```typescript
type Snapshot = {
  schema_version: number;          // 現行 1
  created_at: string;              // RFC 3339
  git_sha: string;                 // 40-char full SHA
  git_action: string;              // "commit" / "revert" / "commit (initial)" 等
  branch: string | null;           // detached HEAD では null
  files: string[];                 // commit で変わったファイル（リポジトリ相対）
  target_org: string;
  label: string | null;            // 自動 snapshot は null
  commit_message: string;          // commit subject（先頭1行）。schema v1 後付け、
                                   // 旧 snapshot は default で空文字列
};
```

## `ManifestEntry` 型（共通）

```typescript
type ManifestEntry = {
  metadata_type: MetadataType;     // 列挙値（下記）
  full_name: string;               // package.xml の <members> に入る値
};

type MetadataType =
  | "ApexClass"
  | "ApexTrigger"
  | "LightningComponentBundle"
  | "CustomObject"
  | "CustomField"
  | "ValidationRule"
  | "Flow"
  | "Layout"
  | "PermissionSet"
  | "Profile"
  | "Other";
```

`full_name` の形式:
- 通常: `"Foo"`（ApexClass）
- object 階層: `"Account.Email__c"`（CustomField / ValidationRule）
- LWC: bundle ディレクトリ名（`"myButton"`）
