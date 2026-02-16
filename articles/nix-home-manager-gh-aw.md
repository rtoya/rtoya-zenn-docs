---
title: "Nix home-managerでgh-aw (GitHub Agentic Workflows) を宣言的に管理する"
emoji: "🤖"
type: "tech"
topics: ["nix", "homemanager", "github", "githubcopilot"]
published: true
publication_name: "atrae"
---

## gh-awとは

`gh-aw` はGitHub公式のAgentic Workflows CLI。GitHub Copilot agentをターミナルから操作できる。

:::message
現在テクニカルプレビュー中のため、まだnix側でサクッとインストールできなかった。
通常は `gh extension install` でインストールできるのでそちらで対応する方が良い。
:::

```bash
gh extension install github/gh-aw
```

本記事ではNix home-managerで宣言的に管理する方法を紹介する。

## やりたいこと

- `gh-aw` をNixで宣言的に管理したい
- `programs.gh.extensions` に登録して他のgh extensionと一元管理したい
- nixpkgsに入っていないパッケージなので、カスタムデリベーションを書く必要がある

## 実装

`home.nix` の `let` ブロックにカスタムデリベーションを追加し、`programs.gh.extensions` に登録する。

```nix
let
  gh-aw = pkgs.stdenv.mkDerivation rec {
    pname = "gh-aw";
    version = "0.45.0";
    src = pkgs.fetchurl {
      url = "https://github.com/github/gh-aw/releases/download/v${version}/darwin-arm64";
      sha256 = "sha256-ftP+aClD91LS7weQYvRV/+6bA43lStbpWNwEQMWNfkk=";
    };
    dontUnpack = true;
    installPhase = ''
      mkdir -p $out/bin
      cp $src $out/bin/gh-aw
      chmod +x $out/bin/gh-aw
    '';
  };
in
{
  programs.gh = {
    enable = true;
    extensions = [ pkgs.gh-dash gh-aw ];
  };
}
```

## ポイント

### fetchurl + pre-built binary

ソースビルド (`buildGoModule`) ではなく、GitHub Releasesのpre-builtバイナリを直接取得している。nixpkgsに入ったら差し替えればよい。

### dontUnpack = true

`gh-aw` のリリースは単体バイナリなのでアーカイブ展開をスキップする。`fetchurl` はデフォルトでアーカイブを展開しようとするため、このフラグがないとビルドが失敗する。

### overlayではなくlet binding

パッケージ1つのためにoverlayを導入するのはオーバーエンジニアリング。`let` に書くのが最もシンプル。

## 反映方法

```bash
nix run github:nix-community/home-manager -- switch --flake .#mbp
gh aw --help
```

## まとめ

nixpkgsに存在しないgh extensionでも、`fetchurl` + `mkDerivation` で簡単にNix管理できる。宣言的に管理することで、環境の再現性が保たれる。
