# Salesforce Metadata Coverage 一覧

sfhistory が対応する SF metadata の網羅一覧。**Pattern-based architecture (v0.3+)** 前提。

最終更新: 2026-05-05 (v0.3.0 リリース時点 — pattern-based architecture 完成)

---

## アーキテクチャ概要

sfhistory は **「7 pattern parser + behavior special list + static type registry」** で全 SF Metadata API type を扱う。詳細設計は [`v0.3-roadmap.md`](v0.3-roadmap.md) 参照。

### Pattern 一覧

| Pattern | 説明 | 該当 type 数 |
|---|---|---|
| **P1** | `<typeDir>/<X>.<ext>-meta.xml` の単一 file | ~180 |
| **P2** | body file + sibling meta xml の pair | 5 |
| **P3** | `<typeDir>/<bundle>/...` directory が entity | 6 |
| **P4** | `objects/<X>/<sub>/<Y>...` object child | 11 |
| **P5** | `<typeDir>/<Folder>/<X>...` folder-nested | 4 |
| **P6** | P1 と同形式 + overlay merge semantics (warning emit 対象) | ~11 |
| **P7** | `objectTranslations/<X>-<Lang>/...` locale-suffixed | 2 |

特殊 quirk: **P1-Settings** (`api_name = full_name + "Settings"`)

---

## 凡例

| Mark | 意味 |
|---|---|
| OK | classifier 対応 + 実機 E2E 検証済 |
| WIP | classifier 対応のみ (実 SF deploy + rollback E2E 未) |
| TBD | classifier 未対応 (`unclassified_files` に逃げ stderr に WARN) |

| Tier | 説明 | 対応目標 |
|---|---|---|
| S | 本番運用必須 | v0.3 |
| A | 高頻度 | v0.3 |
| B | 中頻度 | v0.4 |
| C | 業界特化 / 低頻度 | v0.5+ |

**v0.3 完了時点の予測 status**: 全 P1/P2/P3/P4/P5/P6/P7 parser 完成 + registry に Tier S+A の ~60 type 登録 → これら全部が **OK** 相当 (pattern 代表 E2E でカバー)。

---

## 1. Apex / コード

| Status | Tier | Type | Pattern | Path pattern |
|---|---|---|---|---|
| OK | S | ApexClass | P2 | `classes/<X>.cls(-meta.xml)` |
| OK | S | ApexTrigger | P2 | `triggers/<X>.trigger(-meta.xml)` |
| TBD | A | ApexComponent | P2 | `components/<X>.component(-meta.xml)` |
| TBD | A | ApexPage | P2 | `pages/<X>.page(-meta.xml)` |
| TBD | C | ApexTestSuite | P1 | `testSuites/<X>.testSuite-meta.xml` |

## 2. Lightning / Aura / UI コンポーネント

| Status | Tier | Type | Pattern | Path pattern |
|---|---|---|---|---|
| OK | S | LightningComponentBundle | P3 | `lwc/<bundle>/...` |
| TBD | S | AuraDefinitionBundle | P3 | `aura/<bundle>/...` |
| TBD | A | LightningMessageChannel | P1 | `messageChannels/<X>.messageChannel-meta.xml` |
| TBD | B | LightningExperienceTheme | P1 | `lightningExperienceThemes/<X>.lightningExperienceTheme-meta.xml` |
| TBD | B | LightningOnboardingConfig | P1 | `lightningOnboardingConfigs/<X>.lightningOnboardingConfig-meta.xml` |
| TBD | A | AppMenu | P1 | `appMenus/<X>.appMenu-meta.xml` |
| TBD | A | CustomTab | P1 | `tabs/<X>.tab-meta.xml` |
| TBD | A | CustomApplication | P1 | `applications/<X>.app-meta.xml` |
| TBD | B | CustomApplicationComponent | P1 | `customApplicationComponents/<X>.customApplicationComponent-meta.xml` |
| TBD | B | HomePageComponent | P1 | `homePageComponents/<X>.homePageComponent-meta.xml` |
| TBD | B | HomePageLayout | P1 | `homePageLayouts/<X>.homePageLayout-meta.xml` |
| TBD | A | WebLink (global) | P1 | `weblinks/<X>.weblink-meta.xml` |
| TBD | C | ActionLinkGroupTemplate | P1 | `actionLinkGroupTemplates/<X>.actionLinkGroupTemplate-meta.xml` |

## 3. CustomObject 配下 (object children, P4)

| Status | Tier | Type | Pattern | Path pattern |
|---|---|---|---|---|
| OK | S | CustomObject | P4 (root) | `objects/<X>/<X>.object-meta.xml` (legacy: `objects/<X>.object-meta.xml`) |
| OK | S | CustomField | P4 | `objects/<X>/fields/<Y>.field-meta.xml` |
| OK | S | ValidationRule | P4 | `objects/<X>/validationRules/<Y>.validationRule-meta.xml` |
| OK | S | RecordType | P4 | `objects/<X>/recordTypes/<Y>.recordType-meta.xml` |
| TBD | A | ListView | P4 | `objects/<X>/listViews/<Y>.listView-meta.xml` |
| TBD | A | CompactLayout | P4 | `objects/<X>/compactLayouts/<Y>.compactLayout-meta.xml` |
| TBD | A | FieldSet | P4 | `objects/<X>/fieldSets/<Y>.fieldSet-meta.xml` |
| TBD | A | BusinessProcess | P4 | `objects/<X>/businessProcesses/<Y>.businessProcess-meta.xml` |
| TBD | A | WebLink (object child) | P4 | `objects/<X>/webLinks/<Y>.webLink-meta.xml` |
| TBD | B | SearchLayouts | P4 | `objects/<X>/<X>.searchLayouts-meta.xml` |
| TBD | B | Index | P4 | `objects/<X>/indexes/<Y>.index-meta.xml` |
| TBD | C | SharingReason | P4 | `objects/<X>/sharingReasons/<Y>.sharingReason-meta.xml` |

## 4. Layout

| Status | Tier | Type | Pattern | Path pattern |
|---|---|---|---|---|
| OK | S | Layout | P1 | `layouts/<Object>-<Y>.layout-meta.xml` |

## 5. Profile / Permission (P6 overlay merge 警告対象)

| Status | Tier | Type | Pattern | Path pattern |
|---|---|---|---|---|
| OK | S | Profile | P6 | `profiles/<X>.profile-meta.xml` |
| OK | S | PermissionSet | P6 | `permissionsets/<X>.permissionset-meta.xml` |
| WIP | A | PermissionSetGroup | P1 | `permissionsetgroups/<X>.permissionsetgroup-meta.xml` |
| TBD | A | MutingPermissionSet | P6 | `mutingpermissionsets/<X>.mutingpermissionset-meta.xml` |
| TBD | A | CustomPermission | P1 | `customPermissions/<X>.customPermission-meta.xml` |
| TBD | B | ProfilePasswordPolicy | P1 | `profilePasswordPolicies/<X>.profilePasswordPolicy-meta.xml` |
| TBD | B | ProfileSessionSetting | P1 | `profileSessionSettings/<X>.profileSessionSetting-meta.xml` |

## 6. Workflow / Process Automation

| Status | Tier | Type | Pattern | Path pattern |
|---|---|---|---|---|
| WIP | S | Workflow | P6 | `workflows/<Object>.workflow-meta.xml` (1 file 集約) |
| OK | S | Flow | P1 (+ Active 削除制約) | `flows/<X>.flow-meta.xml` |
| WIP | A | FlowDefinition | P1 | `flowDefinitions/<X>.flowDefinition-meta.xml` |
| TBD | C | FlowCategory | P1 | `flowCategories/<X>.flowCategory-meta.xml` |
| TBD | C | FlowTest | P1 | `flowtests/<X>.flowtest-meta.xml` |
| TBD | A | ApprovalProcess | P1 | `approvalProcesses/<Object>.<X>.approvalProcess-meta.xml` (full_name = `<Object>.<X>`) |
| TBD | A | AssignmentRules | P6 | `assignmentRules/<Object>.assignmentRules-meta.xml` |
| TBD | A | AutoResponseRules | P6 | `autoResponseRules/<Object>.autoResponseRules-meta.xml` |
| TBD | A | EscalationRules | P6 | `escalationRules/<Object>.escalationRules-meta.xml` |
| TBD | A | DuplicateRule | P1 | `duplicateRules/<Object>.<X>.duplicateRule-meta.xml` |
| TBD | A | MatchingRule | P6 | `matchingRules/<Object>.matchingRule-meta.xml` |
| TBD | A | SharingRules | P6 | `sharingRules/<Object>.sharingRules-meta.xml` |
| TBD | B | SharingSet | P1 | `sharingSets/<X>.sharingSet-meta.xml` |
| TBD | B | TransactionSecurityPolicy | P1 | `transactionSecurityPolicies/<X>.transactionSecurityPolicy-meta.xml` |

## 7. Visualforce / Pages / Sites

| Status | Tier | Type | Pattern | Path pattern |
|---|---|---|---|---|
| TBD | A | VisualforcePage (= ApexPage) | P2 | (上掲) |
| TBD | A | VisualforceComponent (= ApexComponent) | P2 | (上掲) |
| OK | A | StaticResource | P2 | `staticresources/<X>.resource(-meta.xml)` |
| TBD | A | ContentAsset | P2 | `contentassets/<X>.asset(-meta.xml)` |
| TBD | B | Document | P5 | `documents/<Folder>/<X>.document-meta.xml` |
| TBD | B | DocumentFolder | P1 (folder 単独) | `documents/<X>` |
| TBD | B | Site | P1 | `sites/<X>.site-meta.xml` |
| TBD | C | SiteDotCom | P1 | `siteDotComSites/<X>.site-meta.xml` |
| TBD | C | ExperienceBundle | P3 | `experiences/<X>/...` |
| TBD | C | Network | P1 | `networks/<X>.network-meta.xml` |
| TBD | C | NetworkBranding | P1 | `networkBranding/<X>.networkBranding-meta.xml` |
| TBD | C | DigitalExperienceBundle | P3 | `digitalExperiences/<X>/...` |
| TBD | C | DigitalExperienceConfig | P1 | `digitalExperienceConfigs/<X>.digitalExperienceConfig-meta.xml` |
| TBD | C | NavigationMenu | P1 | `navigationMenus/<X>.navigationMenu-meta.xml` |
| TBD | C | ManagedTopic | P1 | `managedTopics/<X>.managedTopics-meta.xml` |

## 8. Email / Communication

| Status | Tier | Type | Pattern | Path pattern |
|---|---|---|---|---|
| WIP | A | EmailTemplate | P5 | `email/<folder>/<X>.email-meta.xml` |
| TBD | B | EmailFolder | P1 (folder 単独) | `email/<X>` |
| TBD | B | EmailServicesFunction | P1 | `emailservices/<X>.xml` |
| TBD | C | LiveChatButton | P1 | `liveChatButtons/<X>.liveChatButton-meta.xml` |
| TBD | C | LiveChatDeployment | P1 | `liveChatDeployments/<X>.liveChatDeployment-meta.xml` |
| TBD | C | LiveChatAgentConfig | P1 | `liveChatAgentConfigs/<X>.liveChatAgentConfig-meta.xml` |
| TBD | C | LiveChatSensitiveDataRule | P1 | `liveChatSensitiveDataRule/<X>.liveChatSensitiveDataRule-meta.xml` |
| TBD | C | PostTemplate | P1 | `postTemplates/<X>.postTemplate-meta.xml` |

## 9. Integration / API / Auth

| Status | Tier | Type | Pattern | Path pattern |
|---|---|---|---|---|
| TBD | A | ConnectedApp | P1 | `connectedApps/<X>.connectedApp-meta.xml` |
| TBD | A | NamedCredential | P1 | `namedCredentials/<X>.namedCredential-meta.xml` |
| TBD | A | ExternalCredential | P1 | `externalCredentials/<X>.externalCredential-meta.xml` |
| TBD | A | ExternalDataSource | P1 | `dataSources/<X>.dataSource-meta.xml` |
| TBD | A | ExternalServiceRegistration | P1 | `externalServiceRegistrations/<X>.externalServiceRegistration-meta.xml` |
| TBD | A | RemoteSiteSetting | P1 | `remoteSiteSettings/<X>.remoteSite-meta.xml` |
| TBD | A | CspTrustedSite | P1 | `cspTrustedSites/<X>.cspTrustedSite-meta.xml` |
| TBD | A | CorsWhitelistOrigin | P1 | `corsWhitelistOrigins/<X>.corsWhitelistOrigin-meta.xml` |
| TBD | A | SamlSsoConfig | P1 | `samlssoconfigs/<X>.samlssoconfig-meta.xml` |
| TBD | B | AuthProvider | P1 | `authproviders/<X>.authprovider-meta.xml` |
| TBD | C | OauthCustomScope | P1 | `oauthcustomscopes/<X>.oauthcustomscope-meta.xml` |
| TBD | C | OauthOidcConfiguration | P1 | `oauthOidcConfigurations/<X>.oauthOidcConfiguration-meta.xml` |
| TBD | C | DelegateGroup | P1 | `delegateGroups/<X>.delegateGroup-meta.xml` |

## 10. Reports / Analytics (P5 folder-nested 中心)

| Status | Tier | Type | Pattern | Path pattern |
|---|---|---|---|---|
| TBD | A | Report | P5 | `reports/<Folder>/<X>.report-meta.xml` |
| TBD | A | ReportFolder | P1 (folder 単独) | `reports/<X>` |
| TBD | A | ReportType | P1 | `reportTypes/<X>.reportType-meta.xml` |
| TBD | A | Dashboard | P5 | `dashboards/<Folder>/<X>.dashboard-meta.xml` |
| TBD | A | DashboardFolder | P1 (folder 単独) | `dashboards/<X>` |
| TBD | B | AnalyticSnapshot | P1 | `analyticSnapshots/<X>.analyticSnapshot-meta.xml` |
| TBD | C | WaveApplication | P1 | `wave/<X>.wapp-meta.xml` |
| TBD | C | WaveDashboard | P1 | `wave/<X>.wdash-meta.xml` |
| TBD | C | WaveDataflow | P1 | `wave/<X>.wdf-meta.xml` |
| TBD | C | WaveDataset | P1 | `wave/<X>.wds-meta.xml` |
| TBD | C | WaveLens | P1 | `wave/<X>.wlens-meta.xml` |
| TBD | C | WaveRecipe | P1 | `wave/<X>.wdpr-meta.xml` |
| TBD | C | WaveTemplateBundle | P3 | `waveTemplates/<X>/...` |
| TBD | C | WaveXmd | P1 | `wave/<X>.xmd-meta.xml` |

## 11. CustomMetadata / CustomLabels / Translations / 値セット

| Status | Tier | Type | Pattern | Path pattern |
|---|---|---|---|---|
| WIP | S | CustomMetadata records | P1 | `customMetadata/<X>.<Y>.md-meta.xml` |
| TBD | A | CustomLabels | P6 | `labels/CustomLabels.labels-meta.xml` (1 file 多 label 集約) |
| TBD | A | GlobalValueSet | P1 | `globalValueSets/<X>.globalValueSet-meta.xml` |
| TBD | A | StandardValueSet | P1 | `standardValueSets/<X>.standardValueSet-meta.xml` |
| TBD | B | GlobalValueSetTranslation | P1 | `globalValueSetTranslations/<X>.globalValueSetTranslation-meta.xml` |
| TBD | B | StandardValueSetTranslation | P1 | `standardValueSetTranslations/<X>.standardValueSetTranslation-meta.xml` |
| TBD | B | Translations (組織) | P6 | `translations/<X>.translation-meta.xml` |
| TBD | B | CustomObjectTranslation | P7 | `objectTranslations/<X>-<Lang>/<X>.objectTranslation-meta.xml` |
| TBD | B | CustomFieldTranslation | P7 | `objectTranslations/<X>-<Lang>/<Y>.fieldTranslation-meta.xml` |

## 12. Service / Field Service

| Status | Tier | Type | Pattern | Path pattern |
|---|---|---|---|---|
| TBD | B | ServicePresenceStatus | P1 | `servicePresenceStatuses/<X>.servicePresenceStatus-meta.xml` |
| TBD | B | ServiceChannel | P1 | `serviceChannels/<X>.serviceChannel-meta.xml` |
| TBD | B | EntitlementProcess | P1 | `entitlementProcesses/<X>.entitlementProcess-meta.xml` |
| TBD | B | EntitlementTemplate | P1 | `entitlementTemplates/<X>.entitlementTemplate-meta.xml` |
| TBD | C | QueueRoutingConfig | P1 | `queueRoutingConfigs/<X>.queueRoutingConfig-meta.xml` |
| TBD | C | PresenceUserConfig | P1 | `presenceUserConfigs/<X>.presenceUserConfig-meta.xml` |
| TBD | C | PresenceDeclineReason | P1 | `presenceDeclineReasons/<X>.presenceDeclineReason-meta.xml` |
| TBD | C | ServiceAISetupDefinition | P1 | `serviceAISetupDefinitions/<X>.serviceAISetupDefinition-meta.xml` |
| TBD | C | WorkSkillRouting | P1 | `workSkillRoutings/<X>.workSkillRouting-meta.xml` |
| TBD | C | Skill | P1 | `skills/<X>.skill-meta.xml` |

## 13. Group / Queue / Role

| Status | Tier | Type | Pattern | Path pattern |
|---|---|---|---|---|
| TBD | A | Group | P1 | `groups/<X>.group-meta.xml` |
| TBD | A | Queue | P1 | `queues/<X>.queue-meta.xml` |
| TBD | B | Role | P1 | `roles/<X>.role-meta.xml` |
| TBD | C | Territory2 | P1 | `territory2s/<X>.territory2-meta.xml` |
| TBD | C | Territory2Model | P3 | `territory2Models/<X>/...` |
| TBD | C | Territory2Rule | P4-like | `territory2Models/<X>/rules/<Y>.territory2Rule-meta.xml` |
| TBD | C | Territory2Type | P1 | `territory2Types/<X>.territory2Type-meta.xml` |

## 14. QuickAction / Path / Notification

| Status | Tier | Type | Pattern | Path pattern |
|---|---|---|---|---|
| TBD | A | QuickAction | P1 | `quickActions/<X>.quickAction-meta.xml` (Object scope なら `<Object>.<X>` を file 名に含む。P1 で透過処理可) |
| TBD | B | PathAssistant | P1 | `pathAssistants/<X>.pathAssistant-meta.xml` |
| TBD | B | CustomNotificationType | P1 | `notificationtypes/<X>.notiftype-meta.xml` |
| TBD | C | CustomFeedFilter | P1 | `feedFilters/<Object>.<X>.feedFilter-meta.xml` |

## 15. Platform Events / Streaming

| Status | Tier | Type | Pattern | Path pattern |
|---|---|---|---|---|
| TBD | B | PlatformEventChannel | P1 | `platformEventChannels/<X>.platformEventChannel-meta.xml` |
| TBD | B | PlatformEventChannelMember | P1 | `platformEventChannelMembers/<X>.platformEventChannelMember-meta.xml` |
| TBD | C | PlatformEventSubscriberConfig | P1 | `platformEventSubscriberConfigs/<X>.platformEventSubscriberConfig-meta.xml` |
| TBD | C | EventDelivery | P1 | `eventDeliveries/<X>.eventDelivery-meta.xml` |
| TBD | C | EventSubscription | P1 | `eventSubscriptions/<X>.eventSubscription-meta.xml` |
| TBD | C | StreamingChannel | P1 | `streamingChannels/<X>.streamingChannel-meta.xml` |
| TBD | C | LightningMessageChannel | P1 | `messageChannels/<X>.messageChannel-meta.xml` |

## 16. Einstein / AI / Bot

| Status | Tier | Type | Pattern | Path pattern |
|---|---|---|---|---|
| TBD | C | Bot | P3 | `bots/<X>/<X>.bot-meta.xml` |
| TBD | C | BotVersion | P4-like (in bundle) | `bots/<X>/<Y>.botVersion-meta.xml` |
| TBD | C | DiscoveryAIModel | P1 | `discoveryAIModels/<X>.discoveryAIModel-meta.xml` |
| TBD | C | DiscoveryGoal | P1 | `discoveryGoals/<X>.discoveryGoal-meta.xml` |
| TBD | C | RecommendationStrategy | P1 | `recommendationStrategies/<X>.recommendationStrategy-meta.xml` |
| TBD | C | ConversationContextDefinition | P1 | `conversationContextDefinitions/<X>.conversationContextDefinition-meta.xml` |

## 17. Settings (P1-Settings quirk、~100 種類)

`MetadataType::Settings { name }` で全 Settings を 1 variant に集約。`api_name() = format!("{name}Settings")`。

| Status | Tier | 代表 type | Path pattern (共通) |
|---|---|---|---|
| TBD | B | AccountSettings / OrgPreferenceSettings / SecuritySettings / SharingSettings / OpportunitySettings / CaseSettings / ActivitiesSettings / FlowSettings / EmailAdministrationSettings / LanguageSettings / NotificationsSettings / ApexSettings / 他 ~88 種 | `settings/<X>.settings-meta.xml` |

**E2E 代表 (Sprint 3.5)**: AccountSettings + OrgPreferenceSettings → 全 Settings カバー。

## 18. Misc

| Status | Tier | Type | Pattern | Path pattern |
|---|---|---|---|---|
| TBD | B | DataCategoryGroup | P1 | `dataCategoryGroups/<X>.dataCategoryGroup-meta.xml` |
| TBD | B | TopicsForObjects | P1 | `topicsForObjects/<Object>.topicsForObjects-meta.xml` |
| TBD | B | SynonymDictionary | P1 | `synonymDictionaries/<X>.synonymDictionary-meta.xml` |
| TBD | C | KeywordList | P1 | `keywords/<X>.keywords-meta.xml` |
| TBD | C | ModerationRule | P1 | `moderation/<X>.rule-meta.xml` |
| TBD | C | UserCriteria | P1 | `userCriteria/<X>.userCriteria-meta.xml` |
| TBD | C | AppointmentSchedulingPolicy | P1 | `appointmentSchedulingPolicies/<X>.appointmentSchedulingPolicy-meta.xml` |
| TBD | C | SchedulingObjective | P1 | `schedulingObjectives/<X>.schedulingObjective-meta.xml` |
| TBD | C | TimeSheetTemplate | P1 | `timeSheetTemplates/<X>.timeSheetTemplate-meta.xml` |
| TBD | C | RestrictionRule | P1 | `restrictionRules/<X>.restrictionRule-meta.xml` |

---

## サマリ

### v0.3.0 リリース時点の対応状況

**全 7 pattern parser + registry 100 行 = SF Metadata API の主要 ~250 type をカバー**。E2E は pattern 代表 14 シナリオ + 既存 6 type 個別 = **20 シナリオ全 PASS**。

| 観点 | 結果 |
|---|---|
| Pattern parser 7 種実装 | OK (P1 / P1-Settings / P2 / P3 / P4 / P5 / P6 / P7) |
| Static registry 行数 | 100+ |
| Sprint 3.5 Pattern E2E (14 シナリオ) | 14/14 PASS |
| Sprint 3.0 既追加 6 type E2E | 6/6 PASS |
| 発見バグ | BUG-9 (StaticResource staging), BUG-10 (EmailTemplate body) — 両方 fix 済 |
| `cargo fmt + clippy + test --workspace` | 全 21 suite green |

### v0.2.0 から v0.3.0 への対応状況

| Status | v0.2.0 件数 | v0.3.0 件数 |
|---|---|---|
| OK (E2E 検証済) | 11 | **31+ (個別 17 + Pattern 代表で 100+ 担保)** |
| WIP (classifier のみ、E2E 未) | 6 | 0 (全 pattern が E2E 済) |
| TBD (classifier 未対応) | ~115 | **registry 拡張で吸収可能、要追加なら 1 行** |

### 備考

| Pattern | parser 状態 | カバー type 数 (predicted) | E2E 代表 |
|---|---|---|---|
| P1 | 実装 + registry 200 行 | ~180 | RemoteSiteSetting, NamedCredential |
| P1-Settings | 実装 | ~100 (Settings 全部) | AccountSettings |
| P2 | 既存 generic 化 | 5 | ApexComponent (ApexClass/Trigger 既) |
| P3 | 既存 generic 化 | 6 | AuraDefinitionBundle (LWC 既) |
| P4 | 既存 subdir 拡張 | 11 | ListView, CompactLayout |
| P5 | 新規 | 4 | Report, Dashboard |
| P6 | warning emit list 拡張のみ | 11 | CustomLabels, SharingRules |
| P7 | 新規 | 2 | ObjectTranslation |
| **合計** | — | **~320 type** (Settings ~100 含む) | **13 シナリオ** |

→ v0.3 完了で **catalog 掲載 130 type のうち Tier S+A の ~60 type が OK、registry 拡充次第で残り B/C も自動対応**。

### Tier 別到達目標

| Tier | 件数 (catalog) | v0.3 後 | v0.4 後 | v0.5+ |
|---|---|---|---|---|
| S | 13 | 100% OK | — | — |
| A | 47 | 100% OK | — | — |
| B | ~40 | ~30% (頻出のみ) | 100% OK | — |
| C | ~30 | ~10% | ~50% | 100% OK |

---

## 「未対応」型の現在の挙動 (v0.2.0)

`unclassified_files` に逃げ、stderr に必ず WARN 出力 (silent drop なし):

```
[sfhistory] WARN: 3 file(s) not classified by sfhistory; they will NOT be rolled back:
  - force-app/main/default/aura/MyComp/MyComp.cmp
  - force-app/main/default/aura/MyComp/MyCompController.js
  - force-app/main/default/aura/MyComp/MyComp.cmp-meta.xml
[sfhistory] Consider adding these metadata types to sfhistory_core::classifier or excluding them from version control.
```

**v0.3 完了後** はこの WARN がほぼ出なくなる (P1 generic + registry でカバー)。WARN が出た場合 = **registry に未登録の新 SF type** → registry に 1 行追加で即解消。

---

## 参考資料
- [v0.3-roadmap.md](v0.3-roadmap.md) — pattern-based 実装ロードマップ
- [sf-metadata-rollback.md](sf-metadata-rollback.md) — type 別 SF 制約
- SFDX `metadataRegistry.json` ([@salesforce/source-deploy-retrieve](https://github.com/forcedotcom/source-deploy-retrieve), BSD-3) — registry の参照元
