---
title: "複数人のプロジェクトで導入するためのTIPS"
---

# S3 バックエンドおよび DynamoDB を使った状態ロックの導入

現在の状態では、terraform の設定ファイルがローカルディレクトリにしか生成されておらず、複数人で作業することができません。複数人で成果物を利用できるようにするには、AWS S3 をバックエンドとして設定し、設定ファイルを S3 に出力されるようにします。

また、複数人が同時に作業しようとすると作業が競合し、不正な状態に上書きされるリスクがあるため、AWS DynamoDB テーブルに作業状態を書き込みできるようにします。これにより作業中の人がいた場合は、Snowflake への変更を防止できます。

* https://www.terraform.io/docs/configuration/blocks/backends/index.html
* https://www.terraform.io/docs/backends/types/s3.html

なお、S3 と DynamoDB を利用するためには、以下のような権限を付与した IAM User Credential をローカルで設定しておく必要があります。

（注）筆者は主にシンガポールリージョンを使っていますが、お好きなリージョンを利用ください。また、AWS リソース名は適切な物に変更ください。

* `s3:ListBucket` on `arn:aws:s3:::your-terraform-backend-bucket` 
* `s3:GetObject` on `arn:aws:s3:::your-terraform-backend-bucket/*`
* `s3:PutObject` on `arn:aws:s3:::your-terraform-backend-bucket/*`
* `dynamodb:GetItem` on `arn:aws:dynamodb:ap-southeast-1:*:table/your-terraform-backend-lock`
* `dynamodb:PutItem` on `arn:aws:dynamodb:ap-southeast-1:*:table/your-terraform-backend-lock`
* `dynamodb:DeleteItem` on `arn:aws:dynamodb:ap-southeast-1:*:table/your-terraform-backend-lock`

S3 と DynamoDB に対応した `provider.tf` が以下の通りです。

```
$ cat provider.tf
terraform {
  required_providers {
    snowflake = {
      source  = "chanzuckerberg/snowflake"
      version = "0.20.0"
    }
  }
  backend "s3" {
    bucket         = "your-terraform-backend-bucket"
    key            = "backend/your-sample-project"
    region         = "ap-southeast-1"
    dynamodb_table = "your-terraform-backend-lock"
  }
}

provider "snowflake" {
  username         = "terraform"
  account          = "your_account_${terraform.workspace}"
  region           = "ap-southeast-1"
  # snowflake plugin does not support encrypt version
  private_key_path = "path/to/xx/rsa_key.p8"
  role             = "SYSADMIN"
}
```

なお、ここで `provider "snowflake"` の箇所で、今回のリソース作成に使う Snowflake 側の環境や認証方式について設定しています。

詳しい設定内容は以下を参照ください。

https://registry.terraform.io/providers/chanzuckerberg/snowflake/latest/docs

# workspace を使った環境ごとの可変値の管理

Terraform では、コードを変更せずに複数環境へデプロイするための機能として、workspace という機能が用意されています。

https://www.terraform.io/docs/state/workspaces.html

workspace を利用するには、まず workspace を作成する必要があります。ここでは名前を dev としておきます。

```
$ terraform workspace new dev
$ terraform workspace show
dev
```

先ほどの provider においては　Snowflake アカウント名として
`"your_account_${terraform.workspace}"`と記述しました。これが実際のworkspaceの値を入れた形へ展開されます。例えば、workspace が `dev` の場合は、アカウント名は `your_account_dev` になります。

```
provider "snowflake" {
  username         = "terraform"
  account          = "your_account_${terraform.workspace}"
  region           = "ap-southeast-1"
  # snowflake plugin does not support encrypt version
  private_key_path = "path/to/xx/rsa_key.p8"
  role             = "SYSADMIN"
}
```

また、より高度な使い方としては以下のように環境によって異なる値を一箇所に集約させることが可能です。

例えば、環境ごとに異なるS3バケットやオブジェクトキーからファイルをロードする `stage` を作成したいとします。その場合、環境ごとに可変の設定まとめたファイルとして `locals.tf` を作っておきます。

```
locals {
  env = {
    dev = {
      s3_landing_bucket     = "your-landing-bucket-dev"
      s3_landing_key_prefix = "data/inputs"
      aws_account_id        = "xxx"
    }
    stg = {
      s3_landing_bucket     = "your-landing-bucket-stg"
      s3_landing_key_prefix = "data/inputs"
      aws_account_id        = "xxx"
    }
    prd = {
      s3_landing_bucket     = "your-landing-bucket-prd"
      s3_landing_key_prefix = "data/inputs"
      aws_account_id        = "xxx"
    }
  }
  environment_vars = contains(keys(local.env), terraform.workspace) ? terraform.workspace : "dev"
  workspace        = merge(local.env["dev"], local.env[local.environment_vars])
}
```

`locals.tf` で定義した変数を参照する形で、リソース定義を作成することで、コードを変更することなく環境ごとに設定を変更できます。

例えば、以下は環境ごとに異なる設定を持った `stage` の定義例です。 `stage` の設定については以下のドキュメントを参照ください。https://registry.terraform.io/providers/chanzuckerberg/snowflake/latest/docs/resources/stage

```
resource "snowflake_stage" "your_stage" {
  name                = "your_stage"
  url                 = "s3://${local.workspace["s3_landing_bucket"]}/${local.workspace["s3_landing_key_prefix"]}/your/stage/"
  database            = snowflake_database.your_db.name
  schema              = snowflake_schema.your_schema.name
  storage_integration = snowflake_storage_integration.your_integration.name
```
