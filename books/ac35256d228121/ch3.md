---
title: "Terraform および Snowflake プラグインの導入"
---

ここからは実際にツールを導入する方法を紹介していきます。前述の通り、Snowflake および Terraform そのものは深く紹介しません。深く知りたい方は、別途、各公式ドキュメントを参照ください。

ここからは実際にツールを導入する方法を紹介していきます。前述の通り、Snowflake および Terraform そのものは深く紹介しません。深く知りたい方は、別途、各公式ドキュメントを参照ください。

# Terraform のインストール

CLI のインストール手順は以下を参照ください。

https://learn.hashicorp.com/tutorials/terraform/install-cli

MacOS で brew が使える環境の場合は、インストールが簡単です。

```
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
```

もし OS のパッケージマネジャーを使えない理由がある場合は、ビルド済みバイナリをダウンロードして、PATH を通すだけでも利用できます。

https://www.terraform.io/downloads.html

# Snowflake プラグインのインストール

以下のコマンドで最新版をインストールできます。

https://github.com/chanzuckerberg/terraform-provider-snowflake

```
curl https://raw.githubusercontent.com/chanzuckerberg/terraform-provider-snowflake/main/download.sh | bash -s -- -b $HOME/.terraform.d/plugins
```

# キーペア認証方式の導入

ここからはTerraformでリソースを作成するユーザを作成します。認証方式としてはキーペアを利用します。

人が Web ブラウザでログインして Snowflake を利用する場合、SSO やパスワードおよび MFA を利用する方式が便利ですが、CI と組み合わせて利用する場合、自動化と相性が良い方式にする必要があります。自動化できる方式の中では、パスワード認証よりもキーペア認証方式の方がセキュアです。認証方式全般は以下を参照ください。

https://docs.snowflake.com/en/user-guide/admin-security.html

キーペアの作成方法は以下の通りです。詳細はドキュメントを参照ください。

（注）プラグインが暗号化済みの秘密鍵をサポートしていないので、ここでは平文の秘密鍵を生成しています。秘密鍵を平文のまま扱うのは安全ではありません。Git リポジトリで安全に秘密鍵を共有するには、平文の秘密鍵を AWS KMS などで暗号化した上で共有すると良いでしょう。この場合、デプロイしたい人は暗号化された秘密鍵を復号化する必要があります。

https://docs.snowflake.com/en/user-guide/key-pair-auth.html#step-1-generate-the-private-key

```
# 秘密鍵（平文の PEM フォーマット）を生成する
$ openssl genrsa 2048 | openssl pkcs8 -topk8 -inform PEM -out rsa_key.p8 -nocrypt

# 公開鍵（平文の PEM フォーマット）を生成する
openssl rsa -in rsa_key.p8 -pubout -out rsa_key.pub
```

次に公開鍵をユーザに設定します。以下は `terraform` というユーザを新規作成し、に公開鍵を設定する場合のクエリ例です。この例は簡単のため `SYSADMIN` ロールを設定しますが、チームの方針に合わせて適切なロールを設定してください。

（注）公開鍵の文字列には改行を入れないようにしてください。

```
CREATE USER "terraform"
  DEFAULT_ROLE = SYSADMIN
  SET RSA_PUBLIC_KEY = 'xxxx';
```

#  ローカル環境の構築

ようやくここから Terraform のコードを書いていきます。まずはローカルの環境を設定します。

以下のようなファイルを作成し、初期化コマンドを実行してください。各種ファイルがダウンロードされ、ローカルの環境構築が行われます。

この provider.tf というファイルでは、今回使うプラグインの名前およびバージョンを指定しています。本記事を執筆段階での最新バージョンは `0.20.0`  です。

詳しい設定内容は以下を参照ください。

https://registry.terraform.io/providers/chanzuckerberg/snowflake/latest/docs

```
$ cat provider.tf
terraform {
  required_providers {
    snowflake = {
      source  = "chanzuckerberg/snowflake"
      version = "0.20.0"
    }
  }
}
$ terraform init
```
