---
title: "Github Actions + Terraformを使ったSnowflakeリソース管理のCI/CDパイプラインの構築"
emoji: "❄️"
type: "tech"
topics:
  - "githubactions"
  - "terraform"
  - "db"
  - "snowflake"
  - "dataengineering"
published: true
published_at: "2023-08-15 08:52"
---

データエンジニアの是枝です。

snowflakeユーザーの方、snowflakeのTerraform化には取り組まれていますか？弊社では、snowflakeの各リソースを絶賛Terraforming中です。

Terraformは、HashiCorpによって開発されたオープンソースのInfrastructure as Code（IaC）ツールです。インフラストラクチャの設定、デプロイ、変更の管理に使用されます。有志によってSnowflake用のTerraform providerが開発しており、それを利用してTerraformでSnowflakeを管理することができます。
https://github.com/Snowflake-Labs/terraform-provider-snowflake

snowflakeの以下のようなリソースをTerraform化することができます。
- データベース
- スキーマ
- ユーザ
- ロール
- ウェアハウス
- テーブル
- ビュー

terraformのドキュメント上にはこれ以外のリソースも管理できることが書かれているので確認してみると良いでしょう。
https://registry.terraform.io/providers/Snowflake-Labs/snowflake/latest/docs


またsnowflakeのTerraformingに関してはsnowflakeの公式ハンズオンが用意されています。こちらを利用することで、ローカルPC上でsnowflakeへの認証を設定し、terraform applyを実行してリソース編集をすることが可能です。
https://quickstarts.snowflake.com/guide/terraforming_snowflake/index.html?index=..%2F..index#0

しかしながら、Terraformに関してはローカル操作よりもバージョン管理やチームの生産性向上、セキュリティと監査、手順の一貫性向上の観点から、CI/CDの仕組みに乗せるのが一般的です。Snowflakeのリソースを手動で管理することは、時間と労力がかかるばかりでなく、ヒューマンエラーを引き起こす可能性もあります。

本記事では、ハンズオンで行ったSnowflakeのTerraformingをGithub Actionsに乗せて、リソース管理を自動化するCI/CDパイプラインの構築方法を解説します。ハンズオンの次に是非挑戦してみてください。

（追記）
書いているうちに、「Github Actions + Terraformを使ったSnowflakeリソース管理のCI/CDパイプラインの構築」の域を超えた記事になってきました。前半はタイトルによく一致しますが、後半はFYIでよろしくお願いします。


# ローカルでのリソース変更テスト
ハンズオンを軽くおさらいしておきます。基本的にはユーザーに認証キーを付与→Terraformを実行するというシンプルな流れになります。

snowflakeへの認証に用いるRSAキーを作成する
```bash
$ cd ~/.ssh
$ openssl genrsa 2048 | openssl pkcs8 -topk8 -inform PEM -out snowflake_tf_snow_key.p8 -nocrypt
$ openssl rsa -in snowflake_tf_snow_key.p8 -pubout -out snowflake_tf_snow_key.pub
```

RSA_PUBLIC_KEYを指定したユーザーを作成しておく
```bash
CREATE USER "tf-snow" RSA_PUBLIC_KEY='RSA_PUBLIC_KEY_HERE' DEFAULT_ROLE=PUBLIC MUST_CHANGE_PASSWORD=FALSE;

GRANT ROLE SYSADMIN TO USER "tf-snow";
GRANT ROLE SECURITYADMIN TO USER "tf-snow";
```

ローカルPCにsnowflakeアカウント情報を追加
```bash
$ export SNOWFLAKE_USER="tf-snow"
$ export SNOWFLAKE_PRIVATE_KEY_PATH="~/.ssh/snowflake_tf_snow_key.p8"
$ export SNOWFLAKE_ACCOUNT="YOUR_ACCOUNT_LOCATOR"
$ export SNOWFLAKE_REGION="YOUR_REGION_HERE"
```

追加・変更したいリソースをmain.tfに記述する
```bash
terraform {
  required_providers {
    snowflake = {
      source  = "Snowflake-Labs/snowflake"
      version = "~> 0.68"
    }
  }
}

provider "snowflake" {
  role = "SYSADMIN"
}

resource "snowflake_database" "db" {
  name = "TF_DEMO"
}

resource "snowflake_warehouse" "warehouse" {
  name           = "TF_DEMO"
  warehouse_size = "large"
  auto_suspend   = 60
}
```

あとはTerraformを実行する
```bash
terraform init
terraform plan
terraform apply
```

これでローカルからのsnowflakeのリソース変更は完了です。実際にsnowflakeのUIにアクセスしてみてTF_DEMOのDBが作られているのを確認してみてください。

ハンズオンの内容はそのままローカルでのリソース変更テストに役立ちます。一度ローカルでsnowflakeのリソースが変更できることが確認できてから、Github Actionsでのリソース変更を行う運用が一般的かと思われます。そのため、snowflakeのTerraform管理を検討している人は、まずハンズオンをしっかりこなしましょう。

# Github Actions の設定
ここからGithub Actionsの設定をしていきます。今回は、「Terraform Plan and Apply on Main Branch Push」という名前のワークフローが、mainブランチに対してpushが行われた際に実行されるように設定していきます。

### Github Secretsの設定
GitHub Secretsは、GitHubリポジトリ内で機密情報を安全に保存し、管理するための仕組みです。snowflakeへの認証を通すために、account情報とpasswordを環境変数化しておき、Github Secretsに登録しておきます。

Github Actionsを設定したいレポジトリに行き、上タブの「Setting」 → 左サイドバーの「Secrets and variables」 → 「Actions」 へと進みます。その後、「New repository secret」ボタンをクリックします。
![](https://storage.googleapis.com/zenn-user-upload/6eddd4fb227d-20230807.png)

SNOWFLAKE_ACCOUNTとSNOWFLAKE_PASSWORDの値を設定してください。
![](https://storage.googleapis.com/zenn-user-upload/46617d778f9e-20230807.png)

ちなみにSNOWFLAKE_ACCOUNTで何を設定すればよいか忘れちゃった人は下記で取得できます。
```sql
SELECT current_account() as YOUR_ACCOUNT_LOCATOR;
```

これでGithub Sercretへの設定は完了です。

### main.ymlの作成
私は.github/workflowsフォルダ階層main.ymlの作成をしています。以下コードの解説をします。

```yml
name: Terraform Plan and Apply on Main Branch Push

on:
  push:
    branches:
      - main

jobs:
  terraform:
    name: 'Terraform Plan'
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: terraform/snowflake
    env:
      SNOWFLAKE_ACCOUNT: ${{ secrets.SNOWFLAKE_ACCOUNT }}
      SNOWFLAKE_PASSWORD: ${{ secrets.SNOWFLAKE_PASSWORD }}

    steps:
    - name: Checkout Repository
      uses: actions/checkout@v3

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v1
      with:
        terraform_version: 1.0.11

    - name: Terraform Initialize
      run: terraform init

    - name: Terraform Validate
      run: terraform validate

    - name: Terraform Plan
      run: terraform plan

    - name: Terraform Apply
      run: terraform apply -auto-approve
```
**トリガー (on)**
このワークフローがいつ実行されるかを定義しています。この場合、main ブランチに対してpushが行われた際に実行されます。

**ジョブ　(jobs)**
jobsセクションで、実行するジョブの定義が行われています。

- ジョブ名 (terraform): 
ジョブの名前を定義しています。
- 実行環境 (runs-on): 
ジョブが実行される仮想環境を定義しています。この場合、最新のUbuntu環境で実行されます。
- defaults: 
各runステップのデフォルトの作業ディレクトリを設定しています。
- 環境変数 (env): 
SNOWFLAKE_ACCOUNTとSNOWFLAKE_PASSWORDの環境変数をGitHubのシークレットから取得しています。

```yml
    steps:
    - name: Checkout Repository
      uses: actions/checkout@v3

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v1
      with:
        terraform_version: 1.0.11
```
- ステップ（steps）：各ステップを定義します。
	- リポジトリのチェックアウト: 
	actions/checkout@v3 を使用して、リポジトリのコードを取得します。
	- Terraformのセットアップ: 
	hashicorp/setup-terraform@v1 を使用して、指定したバージョン（1.0.11）のTerraformをセットアップします。


```yml
    - name: Terraform Initialize
      run: terraform init

    - name: Terraform Validate
      run: terraform validate

    - name: Terraform Plan
      run: terraform plan

    - name: Terraform Apply
      run: terraform apply -auto-approve
```
こちらのコードでは主にTerraformの実行までを示しています。

- Terraformの初期化: 
terraform init コマンドを実行して、Terraformの初期化を行います。このコマンドは、指定されたディレクトリ (terraform/snowflake) 内で実行されます。
- Terraformの検証: 
terraform validate コマンドで、コードの検証を行います。これは、Terraformの構成ファイルに構文上のエラーがないか確認するものです。
- Terraformのプラン: 
terraform plan コマンドで、変更計画を表示します。これにより、適用される変更が何であるかを事前に確認できます。
- Terraformの適用: 
terraform apply -auto-approve コマンドで、実際に構成の変更を適用します。-auto-approve フラグにより、承認プロンプトなしで変更が適用されます。

これで、mainにPRを出してマージするとsnowflakeのリソースが変更できるようになっていると思います。


# 弊社におけるTerraformで管理しているSnowflakeリソース
実際に弊社では以下のsnowflakeリソースをTerraformで管理しています（まだできていないものもあるので予定です）。メジャーどころだけ記載します。

- データベース
- スキーマ
- ステージ
- ロール
- ユーザー
- ウェアハウス
- snowpipe
- stream

etc...

テーブルやビューは管理しないの...?と思う方もいると思いますが、ビジネスユーザーがデータマート層に変更を加えることを考えると、terraform管理を導入することでデータドリブンの文化を阻害してしまう可能性を懸念しました。また弊社では今後dbtの導入を行っている途中で、そちらに責務を移譲する形が良いと考えました。このあたりは個社ごとの開発体制やユースケースに基づいて自社におけるsnowflakeのterraform管理すべきリソースの最適解を探す必要があると思います。以下に参考にさせていただきました記事を掲載します。

https://zenn.dev/dataheroes/articles/2021-12-23-terraform

# SnowflakeリソースにおけるTerraformのstate管理
snowflakeをterraform管理する際にstate管理をすることができます。Terraformのstate管理は、リソースの一貫性とチームのコラボレーションを強化する重要な側面であり、適切に構成されるべきです。

Terraformのstate管理を適切に行わなかった場合、以下のような問題やリスクが生じる可能性があります。

**不一致** stateがリアルタイムのインフラストラクチャと一致していない場合、Terraformは何が実際にデプロイされているかを知らないため、予期しない変更や破壊が発生する可能性があります。

**競合** 複数の開発者またはオペレータが同時にTerraformを実行する場合、stateのロックが行われていないと、stateのオーバーライドや競合が生じる可能性があります。

**データの紛失** ローカルのstateファイルが破損したり、失われたりすると、現在のインフラの状態情報が失われ、復元が難しくなる場合があります。

**セキュリティリスク** stateファイルにはセンシティブな情報（パスワード、アクセスキー、エンドポイントなど）が含まれる場合があり、これが不適切に管理されると、セキュリティリスクとなります。

**ヒューマンエラー** 手動でstateを編集するなどのヒューマンエラーが生じた場合、予期しない問題やインフラの破壊が発生する可能性があります。

**コラボレーションの問題** チームで作業する場合、stateの一貫性や可視性がないと、チーム間のコミュニケーションやコラボレーションが難しくなります。

**効率性の低下** state管理が不適切な場合、変更を適用する前に手動での確認や調整が必要となり、デプロイメントプロセスの効率が低下する可能性があります。

要するに、Terraformのstate管理を適切に行わないと、インフラの安定性、セキュリティ、および運用効率に悪影響が及ぶ可能性があります。適切なstate管理は特に大規模なプロジェクトや複数の人が関与する環境では重要です(逆に一人でやる分にはそこまで問題にならないかもしれません)。

ちなみに弊社ではbackend S3とDynamoDBを用いてリモートstate管理をする環境を整えております。


# AWSリソースとSnowflakeリソースを区別する
ここまでで実際にsnowflakeのリソース変更をGithub ActionsからのTerraform applyによってできるようになったかと思います。
しかしながら、同じレポジトリでGithub Actions + Terraform でAWSなどのクラウドリソースも管理している方もいらっしゃるのではないでしょうか。例えば、DMS→S3→snowflakeのパイプラインを作成済みだとすると、DMS→S3の部分はawsリソースとしてterraform管理しておきたいということもあると思います。

その場合は、Github Actionsの設定をAWSとSnowflakeで分ける必要があります。弊社ではGithubの同一レポジトリ上でAWSとSnowflakeのディレクトリを分けて管理しています。

```yml
├── .github
|   └── workflow
|       └── aws.yml
|       └── snowflake.yml
├── terraform
|   └── aws
|   └── snowflake
```

以上のように.github/workfrowの中で**aws.yml**と**snowflake.yml**を作成し、terraformディレクトリでは**aws**と**snowflake**のディレクトリを作っておき、その中でそれぞれのterraformファイルの管理を行っています。


# 終わりに
いかがだったでしょうか。一番の鬼門はgithubからsnowflakeへの認証を通すところなのかなと感じます（そもそも、今回に限らず認証通すってなかなか難しい作業ですよね）。それでも最新のsnowflakeリソース構成が、Terraformで管理されることは、属人化や保守の側面からも有用です。ぜひ御社でもGithub Actions + Terraformを使ったSnowflakeリソース管理のCI/CDパイプラインの構築をやってみてください！


# 参考
SnowflakeのTerraformingに関してはすでに多くの素晴らしい記事が書かれています。どれもとても参考になりますので是非一度目を通してみてください！


SnowflakeとTerraformで作るデータ基盤入門
https://zenn.dev/mnagaa/books/3d668d2dfc657e
Terraform無しでSnowflakeを始めちゃった人へのTerraform導入ガイド
https://zenn.dev/yamnaku/articles/6f1d45640125b2
