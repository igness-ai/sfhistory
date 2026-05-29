# Branch 戦略

igness-ai 社内ルールに従い、sfhistory は **`dev` / `stg` / `prd`** の 3 環境ブランチで運用する。
**`main` は使わない**。

## 各 branch の役割

| branch | 役割 | merge 元 | デフォルト |
|--------|------|---------|----------|
| **`dev`** | 日常開発のメインライン。実装中・WIP も含む | feature branch / 直 push | ✅ default |
| **`stg`** | 統合検証 (staging) ブランチ。dogfood / QA に使う。**配布チャネルではない** | `dev` からの fast-forward / PR | — |
| **`prd`** | 本番ライン。**`just release-production` はこの branch 上で実行する** | `stg` からの PR（必須レビュー想定） | — |

## 命名規則

- feature 作業: `feat/<topic>` `fix/<topic>` `chore/<topic>` 等を `dev` から切る
- リリース version: `v<semver>`（例: `v0.0.1`、`v0.0.1-rc.1`、`v0.1.0`）

## マージ方向

```
feat/* → dev → stg → prd
                       │
                       └── just release-production X.Y.Z → release artifact (sfhistory)
```

逆方向のマージ（prd → stg → dev）は hotfix 適用後の back-merge のみ。

## 配布の 2 系統

### Production 配布（社外 / 一般公開）

- **エントリポイント**: `just release-production X.Y.Z`（`prd` branch 上で実行必須）
- **対象**: Salesforce 開発者などの外部ユーザ
- **チャネル**: GitHub Releases (`igness-ai/sfhistory`) / Homebrew tap (`igness-ai/homebrew-tap`) / installer.sh / installer.ps1 / npm
- **詳細**: 後述「リリース手順」参照

### Staging 配布（社内 / dogfood）

- **エントリポイント**: 社員が手元で `just install-staging`
- **対象**: igness-ai 社内 dev（Rust toolchain あり前提）
- **チャネル**: ローカル build → `~/.local/bin/sfhistory-stg` に配置
- **GitHub Actions は走らせない**（社内 dogfood のために release artifact を作らない方針）
- **配布対象は社内のみ**（社外向けには staging を出さない）

```bash
# 社員側の運用
git checkout stg                # or dev、検証したい branch
git pull
just install-staging            # ~/.local/bin/sfhistory-stg にビルド済 binary が置かれる
sfhistory-stg --version         # production の sfhistory と共存可能
```

## GitHub Actions trigger 方針

**全 workflow は `workflow_dispatch` のみ**（auto-trigger は意図的に無効化）。

| workflow | trigger | 役割 |
|---------|---------|------|
| `ci.yml` | `workflow_dispatch` のみ | fmt / clippy / test / doc / deny を手動で走らせる。将来 `pull_request: branches: [dev, stg, prd]` を再有効化予定 |
| `audit.yml` | `workflow_dispatch` のみ | cargo-audit を手動で走らせる |
| `release.yml` | `workflow_dispatch` のみ | input.version の Cargo.toml 一致確認 → matrix build → sfhistory (public) に push → Homebrew tap 自動更新 |

**`push: tags` トリガは意図的に外している**。tag を切ったら自動で release が走る方式は事故 release リスクが高い。`just release-production` を経由する**明示判断**でしか release が出ない構造にしている。

## リリース手順

### 通常 feature 作業

```bash
git checkout dev
git checkout -b feat/awesome
# ... 実装 + commit ...
git push -u origin feat/awesome
gh pr create --base dev --head feat/awesome
# review → merge to dev
```

### staging 検証 promote

```bash
gh pr create --base stg --head dev --title "promote: dev → stg"
# merge 後、社員は git checkout stg && git pull && just install-staging で更新
```

### production リリース

```bash
gh pr create --base prd --head stg --title "promote: stg → prd"
# merge 後
git checkout prd && git pull
just release-production 0.0.1     # 0.0.1-rc.1 / 0.1.0 等 semver 形式
# → release-production.sh が:
#   1. semver 検証 / prd branch 必須 / 未コミット変更拒否
#   2. 同名 release 重複拒否 / ダウングレード拒否
#   3. Cargo.toml の 3 セクション (workspace.package + dependencies.sfhistory-{core,git})
#      の version を一括 bump → cargo check で Cargo.lock 同期 → 1 commit
#   4. "production" 入力で明示確認
#   5. git push origin prd
#   6. gh workflow run release.yml -f version=0.0.1
# → release.yml が:
#   1. validate: Cargo.toml と input.version の一致確認
#   2. build: 4 OS matrix (Apple Silicon / Intel Mac / Linux x64 / Windows x64)
#   3. publish: igness-ai/sfhistory (public) に gh release create
#   4. publish-homebrew: formula を render → homebrew-tap に push
```

### prerelease のマーク

`v0.0.x` および `*-rc*` / `*-beta*` / `*-alpha*` 接尾辞を持つ version は release.yml が自動で **Pre-release** マークを付ける。`brew install` の最新版選択や `installer.sh` の latest 解決でデフォルト無視されるため、stable に至るまでは prerelease として扱われる。

## 必須 GitHub Secret

| name | 用途 | 権限 |
|------|------|------|
| `RELEASES_TOKEN` | `sfhistory` (public) への release 作成 + (fallback) `homebrew-tap` への formula push | contents:write (両 repo) |
| `HOMEBREW_TAP_TOKEN` (任意) | `homebrew-tap` への push を分離したい場合 | contents:write (homebrew-tap) |

`HOMEBREW_TAP_TOKEN` 未設定なら `RELEASES_TOKEN` にフォールバックする設計。dogfood 期は単一 PAT で十分。

## 旧 `main` branch

リポジトリ初期化時には `main` で開発を開始したが、社内ルールに合わせ `dev` に rename し、`main` は削除済（履歴は `dev` に完全に保持されている）。
