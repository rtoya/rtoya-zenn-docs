---
title: "WezTermでAWS ARNをクリックしてAWSコンソールを開く設定"
emoji: "🔗"
type: "tech"
topics: ["wezterm", "aws", "terminal"]
published: true
publication_name: "atrae"
---

# 概要

WezTermのhyperlink_rules機能を使って、ターミナルに表示されたAWS ARN（Amazon Resource Name）をCmd+クリックするだけでAWSコンソールの該当リソースページを開けるようにする設定を紹介します。

# 背景

AWSを使った開発・運用では、ターミナルにARNが表示される場面が頻繁にあります。

```
arn:aws:iam::123456789012:role/my-eks-role
arn:aws:s3:::my-bucket
arn:aws:lambda:ap-northeast-1:123456789012:function:my-function
```

これらをコピーしてAWSコンソールで検索するのは手間がかかります。WezTermのhyperlink_rules機能を使えば、ARNをクリックするだけでAWSコンソールを開けるようになります。

# 設定方法

## 基本設定

`~/.wezterm.lua`に以下を追加します。

```lua
local wezterm = require 'wezterm'
local config = wezterm.config_builder()

config.hyperlink_rules = {
  -- AWS ARN をマッチ
  {
    regex = [[arn:aws[a-z0-9-]*:[a-z0-9-]*:[a-z0-9-]*:[0-9]*:[a-zA-Z0-9_./:@=-]+]],
    format = "https://console.aws.amazon.com/go/view?arn=$0",
  },
}

-- デフォルトルール（URL、メールアドレス）を追加
for _, rule in ipairs(wezterm.default_hyperlink_rules()) do
  table.insert(config.hyperlink_rules, rule)
end

return config
```

この設定により、ターミナルに表示されたARNにマウスカーソルを合わせるとハイライトされ、Cmd+クリック（macOS）またはCtrl+クリック（Windows/Linux）でAWSコンソールが開きます。

## AWS Console Go View

`https://console.aws.amazon.com/go/view?arn=<ARN>` はAWSが提供するリダイレクトサービスで、ARNを渡すと該当リソースのコンソールページに自動的にリダイレクトされます。

対応しているリソースタイプの例：
- IAM Role/User/Policy
- S3 Bucket
- Lambda Function
- ECS Service/Task
- RDS Instance
- など多数

# ハマりポイント：ファイルパスルールとの競合

## 問題

ARNには`/`が含まれるため、ファイルパスを検出するルールがあると競合が発生します。

例えば、以下のようなファイルパスルールがある場合：

```lua
table.insert(config.hyperlink_rules, {
  regex = "\\b(/[\\w.-]+)+\\b",
  format = "file://$0",
})
```

ARN `arn:aws:iam::123456789012:role/my-eks-role` の `/my-eks-role` 部分がファイルパスとして認識され、`file:///my-eks-role` として開こうとしてしまいます。

## 原因

WezTermのhyperlink_rulesは**定義順に評価**されます。最初にマッチしたルールが適用されるため、ファイルパスルールがARNルールより先に定義されていると、ARNの一部がファイルパスとしてマッチしてしまいます。

## 解決策

**ARNルールを最初に定義**することで、ARN全体が先にマッチするようにします。

```lua
config.hyperlink_rules = {
  -- AWS ARN を最優先でマッチ（他のルールより先に定義）
  {
    regex = [[arn:aws[a-z0-9-]*:[a-z0-9-]*:[a-z0-9-]*:[0-9]*:[a-zA-Z0-9_./:@=-]+]],
    format = "https://console.aws.amazon.com/go/view?arn=$0",
  },
}

-- デフォルトルールを追加
for _, rule in ipairs(wezterm.default_hyperlink_rules()) do
  table.insert(config.hyperlink_rules, rule)
end

-- ファイルパスルールはARNルールの後に追加
table.insert(config.hyperlink_rules, {
  regex = "\\b(/[\\w.-]+)+\\b",
  format = "file://$0",
})
```

# 完成版の設定例

複数のカスタムルールを含む完成版の設定例です。

```lua
local wezterm = require 'wezterm'
local config = wezterm.config_builder()

-- ============================================
-- Hyperlink Rules（クリック可能なリンク）
-- ============================================
-- 注意: ルールは先に定義されたものが優先される

config.hyperlink_rules = {
  -- AWS ARN を最優先でマッチ
  {
    regex = [[arn:aws[a-z0-9-]*:[a-z0-9-]*:[a-z0-9-]*:[0-9]*:[a-zA-Z0-9_./:@=-]+]],
    format = "https://console.aws.amazon.com/go/view?arn=$0",
  },
}

-- デフォルトルール（URL、メールアドレス）を追加
for _, rule in ipairs(wezterm.default_hyperlink_rules()) do
  table.insert(config.hyperlink_rules, rule)
end

-- ファイルパスをクリック可能に（ARNルールより後）
table.insert(config.hyperlink_rules, {
  regex = "\\b(/[\\w.-]+)+\\b",
  format = "file://$0",
})

-- GitHub issue/PR参照（例: owner/repo#123）
table.insert(config.hyperlink_rules, {
  regex = [[\b([A-Za-z0-9_-]+/[A-Za-z0-9_-]+)#(\d+)\b]],
  format = "https://github.com/$1/issues/$2",
})

return config
```

# 動作確認

設定後、WezTermを再起動（または設定をリロード）して以下のコマンドで確認できます。

```bash
echo "arn:aws:iam::123456789012:role/my-eks-role"
echo "arn:aws:s3:::my-bucket"
echo "arn:aws:lambda:ap-northeast-1:123456789012:function:my-function"
```

ARN全体がハイライトされ、クリックするとAWSコンソールが開けば成功です。

# デバッグ方法

設定が正しく適用されているか確認するには、WezTermのDebug Overlay（Ctrl+Shift+L）を開き、以下を入力します。

```lua
window:effective_config().hyperlink_rules
```

これで現在有効なhyperlink_rulesの一覧が確認できます。

# まとめ

WezTermのhyperlink_rules機能を使うことで、ARNをワンクリックでAWSコンソールに遷移できるようになり、AWS運用の効率が向上します。

ポイントは**ルールの定義順序**です。ARNルールを最初に定義することで、ファイルパスルールとの競合を避けられます。

# 参考

- [WezTerm hyperlink_rules](https://wezterm.org/config/lua/config/hyperlink_rules.html)
- [WezTerm Hyperlinks](https://wezterm.org/hyperlinks.html)
- [WezTermでAWS ARNをクリックしてマネジメントコンソールを開く](https://zenn.dev/thinaticsystem/articles/2e15210d3be34c)
- [WeztermでARNのページを開く](https://zenn.dev/mozumasu/articles/mozumasu-aws-tips#wezterm%E3%81%A7%E9%81%B8%E6%8A%9E%E3%81%97%E3%81%9Farn%E3%81%AE%E3%83%9A%E3%83%BC%E3%82%B8%E3%82%92%E9%96%8B%E3%81%8F)
