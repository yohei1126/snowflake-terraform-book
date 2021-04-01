---
title: "はじめに"
---

# 本書の概要

本書は Infrastructure as Code（インフラのコード化）のツールとして人気のある Terraform を使って、テーブルやビューといった Snowflake のリソースのデプロイを自動化を行う方法について紹介します。

本書で紹介する Snowflake プラグインはサードパーティが開発・公開し、OSS コミュニティで開発が行われているため、Snowflake 社の公式ツールではありません。ただし、Terraform のプラグインであるため、CLIで実行できる、デプロイの前に差分確認ができるなど、公式ツール（Web UI、CLI、Python API など）に比べ、構成管理とデプロイの自動化と相性が良いツールだと考えられます。

https://github.com/chanzuckerberg/terraform-provider-snowflake

# 著者のプロフィール

2007 年から 10 年近く東京で日系 SIer や小売企業でソフトウェアエンジニアとして働いた後、2018 年にシンガポールへ移住しました。現在は FinTech 企業で Data Engineer として働いています。普段は、データウェアハウスやデータパイプラインの構築を行なっています。

# 想定読者

* データウェアハウスとして社内に Snowflake を導入し、管理を行なっている方
* DevOps に取り組んでいる方

# 前提

本書では、Snowflake び 自体
