---
title: "基本的なリソースの作成"
---

# その他のリソース作成

データベース、スキーマ、テーブル、ロールなど主要な Snowflake のリソースは、Terraform を使って作成できます。各リソースの定義方法は以下を参照ください。

https://registry.terraform.io/providers/chanzuckerberg/snowflake/latest/docs/resources/table

なお、テーブルの場合は以下のように記述します。

```
resource snowflake_table table {
  database = "database"
  schema   = "schmea"
  name     = "table"
  comment  = "A table."
  owner    = "me"

  column {
    name = "id"
    type = "int"
  }

  column {
    name = "data"
    type = "text"
  }
}
```

# 差分確認およびデプロイ

各種リソースが定義できたところで、実際にデプロイしましょう。一般的な手順は、　`plan` で差分を確認し、 `apply` で実際に変更を反映します。

なお、S3 バックエンドを使った場合は、`plan` の段階でバックエンド上の状態をロックします。タイムアウトするまでは他の人が状態を変更できなくなります。

```
# workspace を選択する
$ terraform workspace select dev
$ terraform init
# 実行計画を作成し、現状との差分を取る
$ terraform plan --out "out.tfplan"
# 問題なければデプロイする
$ terraform apply "out.tfplan"
```

一般的なワークフローとコマンドについては以下を参照ください。

https://www.terraform.io/docs/cli/run/index.html