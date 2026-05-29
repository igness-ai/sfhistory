# Blaze CLI: SessionEnd / PreCompact / SessionStart hook 追加仕様

## 背景

sfhistory が 4 hook (Stop / SessionEnd / PreCompact / SessionStart) 体制でデータロス防止を実装した。Blaze CLI 側は現在 **Stop hook のみ** 実装済み。残り 3 event を追加することで、`/clear` / `/compact` / 異常終了からの起動時リカバリが Claude Code と同等に動く。

Codex 用 task card。`/Users/OguraTakuma/Downloads/BlazeAI/blaze-cli/` 配下で作業。

## 現状 (調査済)

| ファイル | 現状 | 変更要否 |
|---|---|---|
| `src/hooks/executor.rs` | `fire_stop_hook` ハードコード、payload に `hook_event_name: "Stop"` 固定 | **要変更** (event 名一般化) |
| `src/hooks/loader.rs` | `load_stop_hooks` で `hooks.Stop[]` 専用に読み込み | **要変更** (event 名引数化) |
| `src/slash/mod.rs` | `/clear` / `/compact` の slash command 定義あり、ただし hook 発火なし | **要変更** (発火追加) |
| `src/tui/app.rs` | `fire_stop_hook` 呼び出しのみ。SessionStart / SessionEnd 相当の呼び出しなし | **要変更** (起動 / 終了で発火) |

## 追加実装

### 1. `src/hooks/loader.rs` を event 名引数化

```rust
// 旧: pub fn load_stop_hooks() -> Vec<StopHook>
// 新: pub fn load_hooks(event_name: &str) -> Vec<StopHook>
```

`hooks_obj.get(event_name)` を返す。 `load_stop_hooks` は呼び出し側ですべて置換 (deprecated alias は残さない、コード量最小原則)。

### 2. `src/hooks/executor.rs` を一般化

```rust
// 新 API
pub async fn fire_hook(event_name: &str, session_id: &str, cwd: &Path);

// payload は hook_event_name を引数で受ける
let payload = json!({
    "session_id": session_id,
    "transcript_path": ...,
    "cwd": cwd.to_string_lossy(),
    "hook_event_name": event_name,
});
```

既存 `fire_stop_hook` は **削除**。呼び出し側 (app.rs:3162, 3194) を `fire_hook("Stop", ...)` に置換。

### 3. `src/slash/mod.rs` で /clear / /compact ハンドラから hook 発火

`/clear` 実行時:
```rust
crate::hooks::executor::fire_hook("SessionEnd", &session_id, &cwd).await;
```

`/compact` 実行時:
```rust
crate::hooks::executor::fire_hook("PreCompact", &session_id, &cwd).await;
```

`/compact` は **圧縮の直前**に発火する必要がある。圧縮処理を呼ぶ前に hook を await すること。

### 4. `src/tui/app.rs` で起動時 / 終了時 hook 発火

**起動時** (post_first_draw_hooks のタイミング or 初回 session load 直後):
```rust
fire_hook("SessionStart", &self.current_session.id, &self.working_dir).await;
```

**終了時** (TUI shutdown handler / Drop 相当):
```rust
fire_hook("SessionEnd", &self.current_session.id, &self.working_dir).await;
```

shutdown ハンドラが await 不能なら、`tokio::runtime::Handle::current().block_on(...)` で同期化。`SIGINT` (Ctrl+C) でも発火するよう、ctrl_c handler から呼ぶこと。

## テスト (TDD 必須)

t_wada 流。各 step で Red → Green → Refactor。

### tests/hooks/executor_test.rs に追加

1. `fire_hook_passes_event_name_in_payload` — `fire_hook("SessionEnd", ...)` 時に stdin payload の `hook_event_name` が `"SessionEnd"`
2. `fire_hook_loads_event_specific_hooks` — `hooks.SessionEnd[]` に install 済 entry だけ発火し、`hooks.Stop[]` の entry は発火しない

### tests/slash/clear_test.rs (新規)

1. `clear_command_fires_session_end_hook` — `/clear` 実行で SessionEnd hook が 1 回呼ばれる

### tests/slash/compact_test.rs (新規)

1. `compact_command_fires_pre_compact_hook_before_compression` — `/compact` 実行で **圧縮前に** PreCompact hook が呼ばれる

### tests/tui/lifecycle_test.rs (新規)

1. `app_startup_fires_session_start_hook` — app 起動完了で SessionStart hook が 1 回
2. `app_shutdown_fires_session_end_hook` — graceful shutdown で SessionEnd hook が 1 回
3. `ctrl_c_fires_session_end_hook_before_exit` — Ctrl+C で SessionEnd を発火してから exit

## 制約

- **OS**: MacOS / Windows のみサポート。Linux / WSL 対応は不要。
- **i18n**: Blaze の既存 i18n パターン (おそらく `src/i18n/`) に従い、英語 / 日本語両方の表示文言を用意。エラーメッセージ / ログ表示で「hook 発火失敗」などのメッセージを出す場合は両言語必須。
- **コーディング**: Blaze CLI の Cargo.toml / clippy 設定に従う。`unsafe_code` / `unwrap_used` / `panic` deny の前提で書く。
- **後方互換**: 既に `~/.blaze/settings.json` に `hooks.Stop[]` を install しているユーザの設定は壊さない (4 event は **任意追加**)。

## sfhistory 側の対応状態 (本リポジトリ)

- `agent_hook::install_hooks` が 4 event を一括 install する (`SFHISTORY_HOOK_EVENTS` 定数で管理)
- `record-event` は `hook_event_name` で分岐:
  - `Stop`: WAL append (差分のみ)
  - `SessionEnd` / `PreCompact`: WAL flush → floating snapshot
  - `SessionStart`: scan + recovery
- 既存 `~/.blaze/settings.json` に重複 entry がある場合、`agent_hook` の重複判定が command path にも対応したため、`sfhistory enable` 再実行で自動 clean up される
