# 配布チャネルと name reservation 戦略

このドキュメントは sfhistory が closed source であることを前提に、各レジストリでの
**ポリシー準拠**な存在確保方針を整理する。

## 結論サマリ

| レジストリ | sfhistory の方針 | 理由 |
|----------|-----------|-----|
| **GitHub org `igness-ai`** | ✅ 取得済 | 公式 repo の所属。最強の reservation |
| **`igness-ai/sfhistory-internal` repo** | ✅ 取得済 | 実 source 置き場（private） |
| **`igness-ai/homebrew-sfhistory` tap** | 🟡 v0.1.0-rc.1 公開時に作成 | 公式 brew install 経路 |
| **GitHub Releases** | 🟡 tag push で自動生成 | macOS / Linux / Windows バイナリ |
| **crates.io** | ❌ 公開しない | 規約上 placeholder squatting 禁止。closed source なので legit な公開も不可 |
| **npm** | ❌ 公開しない | 同上。sfhistory は JS ツールではない |
| **Homebrew core** | 🔵 v0.1.0 GA + 一定の利用実績後に検討 | 厳格な PR レビュー必要 |
| **nixpkgs** | 🔵 v0.1.0 GA 以降に検討 | 同上 |
| **trademark 登録** | 🔵 商業展開時に検討 | 実効的な法的保護 |

## crates.io / npm を見送る根拠

### 公式ポリシーの引用

[crates.io policy](https://crates.io/policies):
> "exists only to reserve a name for a prolonged period of time (often called
> 'name squatting') **without having any genuine functionality, purpose, or
> significant development activity** on the corresponding repository"

→ プレースホルダ目的の publish は明確に **規約違反**。crates.io 運営は
　該当 crate を予告なく削除できる。

[npm policies](https://docs.npmjs.com/policies/disputes/):
> "Accounts violating the name squatting policy may be removed or renamed without notice."

→ npm も同様。

### "Wrapper-downloader" パターンは可能だが not worth it

`esbuild` / `prettier` などは、open-source な wrapper（npm パッケージ）が
インストール時に native バイナリを GitHub Releases からダウンロードして実行する。
これは「genuine functionality」を持つので squatting にあたらない。

sfhistory でも同じパターンは可能だが:

- sfhistory の **target audience（日本の SF 開発者）は brew からインストールする**。
  npm / crates.io 経由のインストールはほぼ発生しない。
- wrapper crate / package は別途 OSS としてメンテが必要。
- バイナリ更新ごとに wrapper のバージョン bump も必要（二重リリース）。

→ **コスト > リターン**。skip が合理的。

### 万が一の squatting 対応

第三者が `sfhistory` を crates.io / npm に公開した場合:

1. **規約違反通報**（squatting policy / 商標侵害）
2. **公式 channel（brew）への誘導を README で徹底**
3. **公式ドメイン取得時は noindex 等も検討しない**（むしろ偽物との差別化を強化）

## 公式 reservation 方法

### 1. GitHub org `igness-ai` ✅

すでに `https://github.com/igness-ai/sfhistory` を取得済。これが**最強の reservation**。

### 2. `igness-ai/homebrew-sfhistory` tap 🟡

v0.1.0-rc.1 の release artifact が確定し sha256 が分かった時点で作成:

```bash
# 1. 別 repo を作成
gh repo create igness-ai/homebrew-sfhistory --public \
  --description "Homebrew tap for sfhistory (Salesforce Time Machine)"

# 2. Formula/sfhistory.rb を作成して push
mkdir homebrew-sfhistory && cd homebrew-sfhistory
git init -b main
mkdir Formula
# (Formula/sfhistory.rb の中身は release artifact の sha256 確定後に埋める)
```

ユーザは `brew tap igness-ai/sfhistory && brew install sfhistory` で取得可能になる。

### 3. domain 取得（推奨、user 作業）

sfhistory.dev / sfhistory.sh / sfhistory.io など。価値順:

- **sfhistory.dev** — Rust dev エコシステムで自然
- **sfhistory.sh** — CLI ツールに自然
- **sfhistory.app** — closed source プロダクトに自然

### 4. 商標登録（v0.1.0 以降、商業展開時）

日本商標 (J-PlatPat) と USPTO の両方を取得すると効果が大きい。
カテゴリは `Class 9: Computer software` 等。

## Homebrew tap formula の雛形

v0.1.0-rc.1 の release が GitHub Actions で生成された後、各 OS の
sha256 を `release artifacts` の `*.sha256` から拾って埋める:

```ruby
class SfHistory < Formula
  desc "Salesforce Time Machine — git-native rollback for AI-driven development"
  homepage "https://github.com/igness-ai/sfhistory"
  version "0.1.0-rc.1"
  license "Proprietary"

  on_macos do
    if Hardware::CPU.arm?
      url "https://github.com/igness-ai/sfhistory/releases/download/v#{version}/sfhistory-v#{version}-aarch64-apple-darwin.tar.gz"
      sha256 "TBD-after-release"
    else
      url "https://github.com/igness-ai/sfhistory/releases/download/v#{version}/sfhistory-v#{version}-x86_64-apple-darwin.tar.gz"
      sha256 "TBD-after-release"
    end
  end

  on_linux do
    url "https://github.com/igness-ai/sfhistory/releases/download/v#{version}/sfhistory-v#{version}-x86_64-unknown-linux-gnu.tar.gz"
    sha256 "TBD-after-release"
  end

  def install
    bin.install "sfhistory"
  end

  test do
    assert_match "sfhistory #{version}", shell_output("#{bin}/sfhistory --version")
  end
end
```

## チェックリスト（v0.1.0-rc.1 公開時）

- [ ] `git push -u origin main` で main を push（**user 手動**：認可されないため AI は実行不可）
- [ ] `git push origin v0.1.0-rc.1` で tag を push → GitHub Actions が release artifact をビルド
- [ ] release artifact の sha256 を `assets/*.sha256` から取得
- [ ] `igness-ai/homebrew-sfhistory` repo を作成し、上記 formula を push
- [ ] README の brew install 例を実証（fresh macOS で検証）
- [ ] ドメイン（sfhistory.dev 等）の取得検討

## チェックリスト（v0.1.0 GA 時、追加）

- [ ] macOS codesign + notarytool で署名
- [ ] Windows Authenticode 証明書取得 → 署名
- [ ] homebrew-core への PR（一定の利用実績がついた後）
- [ ] nixpkgs への PR（同上）
- [ ] 商標登録（日本 + 米国）
