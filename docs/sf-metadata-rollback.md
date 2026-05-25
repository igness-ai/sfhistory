# Salesforce メタデータ Rollback ガイド

## 対応メタデータ

| メタデータ | rollback |
|---|---|
| Apex (Class / Trigger / Page / Component) | 自動 |
| Lightning (LWC / Aura) | 自動 (bundle単位) |
| StaticResource | 自動 |
| CustomObject / CustomField / ValidationRule | 自動 (ごみ箱から15日間復元可能) |
| RecordType | 追加・変更のみ自動 |
| Layout (変更) | 自動 (XMLコメントは保存されない) |
| CustomMetadata レコード | 自動 |
| EmailTemplate / EmailFolder | 自動 |
| PermissionSetGroup | 自動 |
| Profile / PermissionSet | 自動 (子要素の削除は反映されない) |
| Workflow / SharingRules / AssignmentRules / EscalationRules / AutoResponseRules / MatchingRules | 自動 (子ルールの削除は反映されない) |
| Flow | 新バージョンが非アクティブで作成されます |

上記以外のメタデータにも対応しております (CustomTab / CustomApplication / QuickAction / WebLink / ListView / ApprovalProcess / RemoteSiteSetting / NamedCredential / Group / Queue / ReportType / GlobalValueSet 等)。


## 段階適用

依存関係エラーが出る場合は sync 対象を分割して順次実行してください。

| 操作 | 順序 |
|---|---|
| 追加・変更 | 基礎 → 利用側 (例: `objects` → `flows`) |
| 削除 | 利用側 → 基礎 (例: `flows` → `objects`) |

```bash
sfhistory sync <sha> force-app/main/default/objects --yes
sfhistory sync <sha> force-app/main/default/flows --yes
```