---
title: "mise導入したらIaC開発が快適すぎた"
emoji: "🛠️"
type: "tech"
topics: ["mise", "terraform", "terragrunt", "precommit", "githubactions"]
published: true
---

この記事は [Qiita GitHub Actions Advent Calendar 2025](https://qiita.com/advent-calendar/2025/github-actions) 22日目の記事です。

## はじめに

Terraform/Terragruntでインフラを管理していると、周辺ツールがどんどん増えていきませんか？

- tflint（静的解析）
- trivy（セキュリティスキャン）
- terraform-docs（ドキュメント生成）
- gitleaks（シークレット検出）
- actionlint（GitHub Actions Lint）
- ...etc

私たちのチームでも以下のような問題に悩まされていました。

- チームメンバーごとにツールバージョンがバラバラ
- 「ローカルでは通ったのにCIで落ちた」問題
- 新メンバーの環境構築に時間がかかる
- コードレビューでフォーマットやLintエラーを指摘する不毛さ

そこで **mise + pre-commit + GitHub Actions** の構成を導入したところ、これらの悩みが一気に解消されました。本記事では、その具体的な設定と運用方法を紹介します。

## 全体構成

```
.
├── .mise.toml              # ツールバージョン管理
├── .pre-commit-config.yaml # コミット前の自動チェック
├── .tflint.hcl             # tflint設定
└── .github/
    ├── actions/
    │   └── setup-mise/     # mise セットアップ用 Composite Action
    └── workflows/
        ├── ci.yml          # PRトリガーのCI
        └── _pre-commit.yml # 再利用可能ワークフロー
```

## miseとは

[mise](https://mise.jdx.dev/)（ミーズ）は、複数の開発ツールのバージョンを一元管理できるツールです。

**特徴：**
- asdf互換のプラグインシステム
- Rust製で高速
- `.mise.toml` でプロジェクトごとにバージョンを固定
- `mise exec -- <command>` で指定バージョンのツールを実行

asdfからの移行も容易で、既存の `.tool-versions` もそのまま読み込めます。

## 1. miseによるツールバージョン管理

### .mise.toml の設定

```toml:.mise.toml
[tools]
actionlint = "1.7.9"
gitleaks = "8.30.0"
opentofu = "1.11.2"
pinact = "3.6.0"
pre-commit = "4.5.1"
terraform-docs = "0.21.0"
terragrunt = "0.96.1"
tflint = "0.60.0"
trivy = "0.68.2"
zizmor = "1.18.0"
```

### 各ツールの役割

| ツール | 用途 |
|--------|------|
| opentofu | Terraform互換のIaCエンジン（Terraformのオープンソースフォーク） |
| terragrunt | Terraformのラッパー（DRY化） |
| tflint | Terraform静的解析 |
| trivy | IaCセキュリティスキャン |
| gitleaks | シークレット検出 |
| terraform-docs | READMEの自動生成 |
| actionlint | GitHub Actions Lint |
| zizmor | GitHub Actionsセキュリティ分析 |
| pinact | GitHub Actionsのバージョンピン留め |
| pre-commit | Git hookマネージャー |

### インストール方法

```bash
# miseのインストール（macOS）
brew install mise

# ツール一式をインストール
cd your-project
mise install
```

これだけで、10個以上のツールが正しいバージョンでインストールされます。

## 2. pre-commitによる自動チェック

### なぜpre-commitを使うのか

コミット前に自動でチェックを走らせることで：
- レビュー前に機械的なエラーを排除
- CI待ち時間の削減
- コードレビューで本質的な議論に集中できる

### 設定ファイル

```yaml:.pre-commit-config.yaml
repos:
  # 標準的なチェック
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: check-added-large-files
      - id: check-json
      - id: check-merge-conflict
      - id: check-yaml
      - id: detect-private-key
      - id: end-of-file-fixer
      - id: trailing-whitespace
        args: [--markdown-linebreak-ext=md]

  # ローカルフック（miseで管理するツール）
  - repo: local
    hooks:
      # GitHub Actions Lint
      - id: actionlint
        name: actionlint
        entry: mise exec -- actionlint
        language: system
        files: ^\.github/workflows/.*\.(yml|yaml)$

      # シークレット検出
      - id: gitleaks
        name: Detect hardcoded secrets
        entry: mise exec -- gitleaks protect --verbose --redact --staged
        language: system
        pass_filenames: false

      # GitHub Actionsバージョンピン留め
      - id: pinact
        name: pinact
        entry: mise exec -- pinact run
        language: system
        files: ^\.github/.*\.(yml|yaml)$

      # Terraformドキュメント自動生成
      - id: terraform-docs
        name: terraform-docs
        entry: bash -c 'shopt -s nullglob; for dir in modules/*/; do mise exec -- terraform-docs markdown table --lockfile=false --output-file README.md "$dir"; done'
        language: system
        files: ^modules/.*\.tf$
        pass_filenames: false

      # Terragrunt HCLフォーマット
      - id: terragrunt-hcl-fmt
        name: terragrunt hcl fmt
        entry: mise exec -- terragrunt hcl fmt
        language: system
        files: \.hcl$
        pass_filenames: false

      # tflint
      - id: tflint
        name: tflint
        entry: bash -c 'CONFIG="$(pwd)/.tflint.hcl"; mise exec -- tflint --init > /dev/null 2>&1; mise exec -- tflint --config="$CONFIG" --chdir=modules --recursive && mise exec -- tflint --config="$CONFIG" --chdir=environments --recursive'
        language: system
        files: \.tf$
        pass_filenames: false

      # OpenTofu フォーマット
      - id: tofu-fmt
        name: tofu fmt
        entry: mise exec -- tofu fmt -recursive
        language: system
        files: \.tf$
        pass_filenames: false

      # OpenTofu バリデーション
      - id: tofu-validate
        name: tofu validate
        entry: bash -c 'for dir in modules/*/; do (cd "$dir" && mise exec -- tofu init -backend=false -input=false > /dev/null 2>&1 && mise exec -- tofu validate) || exit 1; done'
        language: system
        files: \.tf$
        pass_filenames: false

      # Trivy セキュリティスキャン
      - id: trivy
        name: trivy IaC scan
        entry: bash -c 'mise exec -- trivy config --exit-code 1 --severity HIGH,CRITICAL --skip-dirs .terragrunt-cache .'
        language: system
        files: \.(tf|hcl)$
        pass_filenames: false

      # GitHub Actionsセキュリティ分析
      - id: zizmor
        name: zizmor
        entry: mise exec -- zizmor
        language: system
        files: ^\.github/.*\.(yml|yaml)$
```

### ポイント：mise exec の活用

すべてのローカルフックで `mise exec --` を使っているのがポイントです。

```yaml
entry: mise exec -- tflint --recursive
```

これにより：
- `.mise.toml` で指定したバージョンのツールが確実に使われる
- グローバルにインストールしたツールと混同しない
- チーム全員が同じバージョンで実行

### tflint設定のカスタマイズ

```hcl:.tflint.hcl
config {
  call_module_type = "local"
}

plugin "terraform" {
  enabled = true
  preset  = "recommended"
}

# Terragruntで管理するためモジュールでは不要
rule "terraform_required_version" {
  enabled = false
}

rule "terraform_required_providers" {
  enabled = false
}
```

### セットアップと実行

```bash
# pre-commitのインストール（初回のみ）
mise exec -- pre-commit install

# 手動で全ファイルチェック
mise exec -- pre-commit run --all-files
```

## 3. GitHub Actionsとの統合

### Composite Actionでmiseをセットアップ

```yaml:.github/actions/setup-mise/action.yml
name: Setup mise
description: Setup mise with caching

inputs:
  cache:
    description: Enable mise and pre-commit cache
    required: false
    default: "true"

runs:
  using: composite
  steps:
    - uses: jdx/mise-action@c37c93293d6b742fc901e1406b8f764f6fb19dac # v2
      with:
        install_args: "--yes"
        cache: ${{ inputs.cache }}

    # pre-commitのキャッシュも設定
    - uses: actions/cache@v4
      if: inputs.cache == 'true'
      with:
        path: ~/.cache/pre-commit
        key: pre-commit-${{ runner.os }}-${{ hashFiles('.pre-commit-config.yaml') }}
```

### CIワークフロー

```yaml:.github/workflows/ci.yml
name: CI
on: pull_request

permissions:
  contents: read

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  pre-commit:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false

      - uses: ./.github/actions/setup-mise

      - name: Run pre-commit
        run: mise exec -- pre-commit run --all-files
```

### ポイント

1. **キャッシュの活用**: mise-actionは自動でツールをキャッシュ。pre-commitの環境も別途キャッシュすることで高速化。

2. **同じチェックの実行**: ローカルの `pre-commit run --all-files` と同じコマンドをCIでも実行。環境差異がなくなる。

3. **Composite Actionで共通化**: setup-miseを再利用可能なActionにすることで、複数ワークフローで一貫した設定を維持。

## 運用してみて

### 導入効果

| Before | After |
|--------|-------|
| 「tflintのバージョンいくつ？」 | `.mise.toml`を見ればわかる |
| CIで初めてエラーに気づく | コミット時点で検出 |
| 新メンバーの環境構築に半日 | `mise install`で数分 |
| レビューでフォーマット指摘 | 自動修正されてコミット |

### Tips

**新しいツールの追加**

```bash
# .mise.tomlの[tools]セクションに追加
# 例: new-tool = "1.0.0"
mise install

# pre-commitにフックを追加
# .pre-commit-config.yamlを編集
```

**特定のフックだけ実行**

```bash
mise exec -- pre-commit run tflint --all-files
mise exec -- pre-commit run trivy --all-files
```

**フックをスキップ（緊急時のみ）**

```bash
git commit --no-verify -m "hotfix"
```

## まとめ

mise + pre-commit + GitHub Actionsの組み合わせで、IaC開発の品質管理を大幅に改善できました。

**得られたメリット：**
- **環境統一**: ローカルとCIで完全に同じツール・バージョン
- **自動化**: コミット前に17種類のチェックが自動実行
- **簡単導入**: 新ツール追加は.mise.tomlに1行書くだけ
- **高速CI**: キャッシュ活用で待ち時間を短縮
- **セキュリティ強化**: trivy、gitleaks、zizmorで多層防御

特にmiseの `mise exec --` による統一実行は、pre-commitとの相性が抜群です。チーム開発でツールバージョンの不一致に悩んでいる方は、ぜひ試してみてください。

## 参考リンク

- [mise 公式ドキュメント](https://mise.jdx.dev/)
- [pre-commit 公式サイト](https://pre-commit.com/)
- [tflint](https://github.com/terraform-linters/tflint)
- [trivy](https://github.com/aquasecurity/trivy)
- [gitleaks](https://github.com/gitleaks/gitleaks)
