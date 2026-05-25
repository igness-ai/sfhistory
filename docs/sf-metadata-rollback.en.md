# Salesforce Metadata Rollback Guide

## Supported Metadata

| Metadata | Rollback |
|---|---|
| Apex (Class / Trigger / Page / Component) | Auto |
| Lightning (LWC / Aura) | Auto (bundle-based) |
| StaticResource | Auto |
| CustomObject / CustomField / ValidationRule | Auto (recoverable from Recycle Bin for 15 days) |
| RecordType | Auto for add and modify |
| Layout (modify) | Auto (XML comments are dropped) |
| CustomMetadata records | Auto |
| EmailTemplate / EmailFolder | Auto |
| PermissionSetGroup | Auto |
| Profile / PermissionSet | Auto (child element deletions are not applied) |
| Workflow / SharingRules / AssignmentRules / EscalationRules / AutoResponseRules / MatchingRules | Auto (child rule deletions are not applied) |
| Flow | New version is created as Inactive |

Many other SF Metadata API types are also auto-rollbacked (e.g. CustomTab, CustomApplication, QuickAction, WebLink, ListView, ApprovalProcess, RemoteSiteSetting, NamedCredential, Group, Queue, ReportType, GlobalValueSet).


## Staged Apply

If a sync fails due to dependencies, split it into multiple invocations by path.

| Operation | Order |
|---|---|
| Add / modify | Foundation → consumer (e.g. `objects` → `flows`) |
| Delete | Consumer → foundation (e.g. `flows` → `objects`) |

```bash
sfhistory sync <sha> force-app/main/default/objects --yes
sfhistory sync <sha> force-app/main/default/flows --yes
```
