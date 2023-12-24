---
title: "Snowpark Container Serviceを触ってみたら革命的すぎた話"
emoji: "❄️"
type: "tech"
topics:
  - "container"
  - "snowflake"
  - "bigdata"
  - "snowpark"
  - "Docker"
published: true
---

本記事は、[Snowflake Advent Calendar 2023](https://qiita.com/advent-calendar/2023/snowflake) の 25 日目です。

データエンジニアの是枝です。

Snowpark Container Service、とうとうGAが始まりましたね...!
Powered by Snowflakeを体現しようと尽力している弊社にとって大変強力な機能です。そこで早速、Tutorialにトライしてみて公式ドキュメントを調査してみたところ、snowflakeをアプリケーション開発に用いている企業にとっては革命的で有ることがわかりました。その記録をこちらの記事で紹介していこうと思います。


::: message alert

2023年12月22日現在、Snowpark Container Services 、AWS の ヨーロッパ (ロンドン)、アジアパシフィック (ムンバイ)でしか使用できません。また、トライアルアカウントはサポートされていないことに注意してください。Azure と GCP のプライベート プレビューは後日発表される予定のようです。こちらは状況が変わり次第、また追記していきます。

https://docs.snowflake.com/en/developer-guide/snowpark-container-services/overview#available-regions

:::
::: message alert
本チュートリアルではDocker Desktopがインストールされている前提で話をしますので、まだの方は[こちら](https://docs.docker.com/get-docker/)を参考にインストールをお願いします。
:::

（追記）
Snowpark Container Serviceに関する記事を一番に出そうとしたら、masuoさんに先越されました...笑
https://zenn.dev/tmasuo/articles/ab21cf8ac1c76f

しかし、Masuoさんの記事はQuick startを取り扱ってるので、私のTutorialといい感じで棲み分けられそうです！両方の記事をご参考ください！！
（GAになってからsnowflake社員以外では一番？）

## Snowpark Container Serviceとは？

公式ドキュメントを日本語訳にかけて要点だけを読んでみます。

https://docs.snowflake.com/en/developer-guide/snowpark-container-services/overview

> Snowpark Container Services は、Snowflake エコシステム内でコンテナ化されたアプリケーションのデプロイ、管理、スケーリングを容易にするように設計されたフルマネージドのコンテナ製品です。このサービスにより、ユーザーはコンテナ化されたワークロードを Snowflake 内で直接実行できるため、処理のためにデータを Snowflake 環境の外に移動する必要がなくなります。Snowpark Container Services は、Snowflake に特化して最適化された OCI ランタイム実行環境を提供します。この統合により、Snowflake の堅牢なデータ プラットフォームを活用して、OCI イメージをシームレスに実行できるようになります。
> 

> Snowpark Container Services は、サードパーティのツールとも統合されています。これにより、サードパーティ クライアント (Docker など) を使用して、アプリケーション イメージを Snowflake に簡単にアップロードできるようになります。
> 

> Snowpark Container Services でコンテナ化されたアプリケーションを実行するには、データベースやウェアハウスなどの基本的な Snowflake オブジェクトの操作に加えて、[イメージ リポジトリ](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/working-with-registry-repository)、 [コンピューティング プール](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/working-with-compute-pool)、[サービス](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/working-with-services)、および*[ジョブ の](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/working-with-jobs)オブジェクトも操作します。
> 

つまり、Snowpark Container Servicesは、Snowflake内で簡単にアプリケーションを実行できるツールです。GPU を用いた LLM を含む機械学習モデルの学習及び実行やあらゆる言語の実行が可能になります。それも外部にデータを移すことなく、Snowflake上で直接アプリを動かせるようになります。

![](https://storage.googleapis.com/zenn-user-upload/cf4d4272e5cf-20231224.png)

> Snowflake は、イメージを保存するための[OCIv2準拠サービスである](https://github.com/opencontainers/distribution-spec/blob/main/spec.md)*イメージ レジストリ*を提供します 。これにより、OCI クライアント (Docker CLI や SnowSQL など) が Snowflake アカウントのイメージ レジストリにアクセスできるようになります。
> 

また、Dockerのような他のツールとも連携して、アプリのアップロードが簡単になります。
普段、コンテナ環境でデプロイされているアプリケーション開発されている方にとってはかなり親和性が高い機能なのではないでしょうか？

## Snowpark Container Servicesのチュートリアルをやってみる
では、早速Snowpark Container Servicesのチュートリアルをやってみようと思います。参考にしたドキュメントはこちらになります。
https://docs.snowflake.com/en/developer-guide/snowpark-container-services/overview-tutorials

## ムンバイまたはロンドンリージョンに環境を作る

チュートリアルを始める前に、orgadminロールを使用して、[create account](https://docs.snowflake.com/ja/sql-reference/sql/create-account)コマンドでムンバイまたはロンドンリージョンに環境を作る必要があります。

ロンドンリージョンで作るなら`aws_eu_west_2` 、ムンバイリージョンでつくるなら`ap-south-1` です。

```sql
use role ORGADMIN;

CREATE ACCOUNT SNOWPARK_CONTAINER_TEST
      ADMIN_NAME = TATSUYA_KOREEDA
      ADMIN_PASSWORD = [passwordを入力]
      EMAIL = '[メールアドレスを入力]'
      EDITION = 'ENTERPRISE'
      REGION = aws_eu_west_2;
```

ちなみにリージョンの識別子が知りたい場合は [SHOW REGIONS](https://docs.snowflake.com/ja/sql-reference/sql/show-regions) で調べることが可能です。

![](https://storage.googleapis.com/zenn-user-upload/c9db8cdfe938-20231222.png)

`SHOW ORGANIZATION ACCOUNTS` のaccount_locator_urlを見ると、ログインURLを調べることができます。新しいアカウント管理者ユーザーは、最初のログイン時にパスワードを変更する必要があります。これで準備完了です。

![](https://storage.googleapis.com/zenn-user-upload/68d6672b35e9-20231222.png)

## Snowflakeの各種オブジェクトを用意する

ACCOUNTADMIN ロールを使用して、usernameをSnowflake ユーザーの名前に置き換えて次のスクリプトを実行します。

```sql
//チュートリアルで使用するロール作成
CREATE ROLE test_role;
GRANT ROLE test_role TO USER <username>;

ALTER USER <username> SET DEFAULT_ROLE = test_role;

//サービスとジョブを実行するコンピューティング プールを作成
CREATE COMPUTE POOL tutorial_compute_pool
  MIN_NODES = 1
  MAX_NODES = 1
  INSTANCE_FAMILY = CPU_X64_XS;

GRANT ALL ON COMPUTE POOL tutorial_compute_pool TO ROLE accountadmin;
GRANT OWNERSHIP ON COMPUTE POOL tutorial_compute_pool TO ROLE test_role;

//使用するウェアハウス作成
CREATE OR REPLACE WAREHOUSE tutorial_warehouse WITH
  WAREHOUSE_SIZE='X-SMALL'
  AUTO_SUSPEND = 180
  AUTO_RESUME = true
  INITIALLY_SUSPENDED=false;

//使用するウェアハウス作成
GRANT ALL ON WAREHOUSE tutorial_warehouse TO ROLE test_role;

//データベース作成
CREATE DATABASE tutorial_db;
//データベースに対する OWNERSHIP権限を付与
GRANT OWNERSHIP ON DATABASE tutorial_db TO ROLE test_role;
```

**`test_role`** ロールがデータベース内に他のリソースを作成してタスクを実行できるようにします。

**`GRANT BIND SERVICE`** に関するドキュメントが見つけられませんでしたが、名前から察するにサービスで定義するエンドポイントにアクセスするための権限を、ロールに付与しているのだと思われます。

```SQL
GRANT BIND SERVICE ENDPOINT ON ACCOUNT TO ROLE test_role;
```

ACCOUNTADMIN ロールを使用して、次のステートメントを実行します。Snowpark Container Servicesでは、認証に Snowflake OAuth を使用します。こちらのコマンドでは新しいOAuthセキュリティ **`snowservices_ingress_oauth`** を作成しています。

```sql
CREATE SECURITY INTEGRATION IF NOT EXISTS snowservices_ingress_oauth
  TYPE=oauth
  OAUTH_CLIENT=snowservices_ingress
  ENABLED=true;
```

データベース内での各種オブジェクトを作成していきます。

```sql
USE ROLE test_role;
USE DATABASE tutorial_db;
USE WAREHOUSE tutorial_warehouse;

CREATE SCHEMA data_schema;
```

その後、**`CREATE OR REPLACE IMAGE REPOSITORY`** にてDockerイメージをアップロードするリポジトリを作成します。

```sql
//リポジトリの作成。Dockerイメージをこのリポジトリにアップロードします。
CREATE OR REPLACE IMAGE REPOSITORY tutorial_repository;
```

そして、ステージを作成し仕様ファイルをアップロードしていきます。各サービスまたはジョブ イメージには、サービスまたはジョブの実行に必要なSnowflake情報を提供する仕様ファイルが含まれています。

```sql
//ステージ作成
CREATE STAGE tutorial_stage DIRECTORY = ( ENABLE = true );
```

`SHOW COMPUTE POOLS` または `DESCRIBE COMPUTE POOL tutorial_compute_pool` でコンピューティング プールの存在を確認してみましょう。

```sql
SHOW COMPUTE POOLS;
DESCRIBE COMPUTE POOL tutorial_compute_pool;
```

きちんとコンピューティング プールの存在を確認できます。

![](https://storage.googleapis.com/zenn-user-upload/9fe5ac2f267d-20231222.png)

## Imageを作成してsnowflakeリポジトリにプッシュする

ここまででSnowpark Container Services上でサービスを作成する準備が整いました。このチュートリアルでは、入力として提供したテキストを単にエコーバックするサービス ( という名前) を作成していきます。

サービスに必要なコードをダウンロードしていきます。今回はsnowflake社が用意しているfileを使用していきますので、[こちら](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/tutorials/tutorial-1#download-the-service-code)にアクセスして、zip fileをクリックしてダウンロードを開始して下さい。

![](https://storage.googleapis.com/zenn-user-upload/592628c45e33-20231222.png)

ターミナルで解凍したファイルが含まれるディレクトリに移動、現在の作業ディレクトリ (.) を指定して次のコマンドを実行します。

```bash
docker build --rm --platform linux/amd64 -t my_echo_service_image:tutorial .
```

image URL ( `<repository_url>/<image_name>`) をタグ付けします。

```bash
docker tag my_echo_service_image:tutorial <repository_url>/my_echo_service_image:tutorial

# 例
# docker tag my_echo_service_image:tutorial \
#  myorg-myacct.registry.snowflakecomputing.com/tutorial_db/data_schema/tutorial_repository/my_echo_service_image:tutorial
```

`<repository_url>`を把握したい場合は、`SHOW IMAGE REPOSITORIES`でrepository_url列を参照してください。

```sql
SHOW IMAGE REPOSITORIES;
```

![](https://storage.googleapis.com/zenn-user-upload/618d3a60c91d-20231222.png)

Snowflake レジストリを使用して Docker を認証していきます。

```bash
docker login <registry_hostname> -u <username>
```

`<registry_hostname>`は SHOW IMAGE REPOSITORIES SQL コマンドを使用してリポジトリ URL として取得されるURLです。URL内のホスト名はレジストリのホスト名です。（例：`myorg-myacct.registry.snowflakecomputing.com`）

`<username>`は Snowflake ユーザー名です。

コマンド入力しますとpasswordが求められますので、Snowflakeユーザーのpasswordを入力すると「Login Succeeded」となります。

それではsnowflake上に作成したリポジトリにイメージをPushしていきましょう。<repository_url>を置き換えて実行してください。

```bash
docker push <repository_url>/my_echo_service_image:tutorial

# 例
# docker push myorg-myacct.registry.snowflakecomputing.com/tutorial_db/data_schema/tutorial_repository/my_echo_service_image:tutorial
```

pushが成功するとターミナルの画面では、以下のようになります。

```bash
(base) t_koreeda@user Tutorial-1 % docker push [レポジトリURL]/tutorial_db/data_schema/tutorial_repository/my_echo_service_image:tutorial
The push refers to repository [レポジトリURL]/tutorial_db/data_schema/tutorial_repository/my_echo_service_image]
0a8710dddce3: Pushed 
ec7c17e51604: Pushed 
77bac83053e8: Pushed 
c5321f7f53ff: Pushed 
df6c1b185b95: Pushed 
b23fedba7dbd: Pushed 
ae2d55769c5e: Pushed 
e2ef8a51359d: Pushed 
tutorial: digest: sha256:a0f8e6eb8f4377e8328938902f51 size: 1996
```

## サービスの作成

次は、[サービスの作成](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/working-with-services)していきます。

以下のコマンドで、`echo_service` というサービスを作成していきます。サービスの仕様は **SPECIFICATION内のYAMLで記述**します。

```sql
CREATE SERVICE echo_service
  IN COMPUTE POOL tutorial_compute_pool
  FROM SPECIFICATION $$
    spec:
      containers:
      - name: echo
        image: /tutorial_db/data_schema/tutorial_repository/my_echo_service_image:tutorial
        env:
          SERVER_PORT: 8000
          CHARACTER_NAME: Bob
        readinessProbe:
          port: 8000
          path: /healthcheck
      endpoints:
      - name: echoendpoint
        port: 8000
        public: true
      $$
   MIN_INSTANCES=1
   MAX_INSTANCES=1;
```

（Service specificationのシンタックスに関しては[こちら](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/specification-reference)にまとまっています。）

YAMLのコードを解説していきます。
- **containers**: コンテナのリストです。**`echo`** という名前のコンテナが定義されており、先程リポジトリにプッシュしたイメージ（**`my_echo_service_image:tutorial`**）から作成されるよう定義します。**`env`** セクションは環境変数を設定し、**`readinessProbe`** はサービスのヘルスチェックに使われる設定です。
- **endpoints**: サービスのエンドポイントを定義します。ここでは **`echoendpoint`** という名前のエンドポイントがポート8000で公開されています。チュートリアルでは、TCP ネットワーク ポートの名前のリストを指定して、public: trueによってPublic endpointを作成しています。

:::message
ちょっと余談ですが、ちなみにステージ情報を使用してサービスを作成する場合は以下の通り。こちらのお陰で、yamlファイルをステージにデプロイするだけで、コンテナ環境を利用できるようになりそうです。

```sql
CREATE SERVICE echo_service
  IN COMPUTE POOL tutorial_compute_pool
  FROM @tutorial_stage
  SPECIFICATION_FILE='echo_spec.yaml';
```

驚いたことに、すでに自動スケーリング+ロード バランサーによるリクエスト分散が実装されており、CREATE SERVICEする際にMIN_INSTANCES プロパティと MAX_INSTANCES プロパティを設定して複数のサービス インスタンスを実行できるようです。

https://docs.snowflake.com/en/developer-guide/snowpark-container-services/working-with-services#create-multiple-service-instances-and-configure-autoscaling

```sql
CREATE SERVICE echo_service
   IN COMPUTE POOL tutorial_compute_pool
   FROM @tutorial_stage
   SPECIFICATION_FILE='echo_spec.yaml'
   MIN_INSTANCES=2
   MAX_INSTANCES=4;
```
:::

## サービス関数の作成

サービスと通信するためのサービス関数を作成していきます。**サービス関数は、サービスとの通信に使用できる方法の1つ**です。サービス エンドポイントに関連付けるユーザー定義関数 (UDF) になります。サービス関数が実行されると、サービス エンドポイントにリクエストが送信され、応答が受信されます。

サービス関数を作成していきます。下記コマンドを実行してください。

```bash
CREATE FUNCTION my_echo_udf (text varchar)
  RETURNS varchar
  SERVICE=echo_service
  ENDPOINT=echoendpoint
  AS '/echo';
```

- SERVICE プロパティは、UDF を`echo_service`サービスに関連付けます。
- ENDPOINT プロパティは、UDF を`echoendpoint`サービス内のエンドポイントに関連付けます。
- AS '/echo' は、Echo サーバーへの HTTP パスを指定します。このパスはサービス コード ( `echo_service.py`) で見つけることができます。

## ****サービスを利用する****

それでは実際に作成したサービスを利用してみましょう。サービス関数の使用ではクエリで呼び出すことができます。サンプルのサービス関数 (`my_echo_udf`) は、単一の文字列または文字列のリストを入力として受け取ることができます。

```bash
SELECT my_echo_udf('hello!');
```

Snowflake は、POST リクエストをサービス エンドポイント ( `echoendpoint`) に送信します。リクエストを受信すると、サービスは応答内の入力文字列をエコーします。

```bash
+--------------------------+
| **MY_ECHO_UDF('HELLO!')**|
|------------------------- |
| Bob said hello!          |
+--------------------------+
```

文字列のリストをサービス関数に渡すと、Snowflake はこれらの入力文字列をバッチ化し、一連の POST リクエストをサービスに送信します。サービスがすべての文字列を処理した後、Snowflake は結果を結合して返します。次の例では、テーブル列を入力としてサービス関数に渡します。

次に複数の文字列を含むテーブルを作成して、サービス関数を呼び出します。テーブルの行を入力として渡して、SELECT ステートメントを実行します。

```bash
CREATE TABLE messages (message_text VARCHAR)AS (SELECT * FROM (VALUES ('Thank you'), ('Hello'), ('Hello World')));
SELECT my_echo_udf(message_text) FROM messages;
```

以下のような結果が返ってきたら成功です！

```bash
+---------------------------+
| MY_ECHO_UDF(MESSAGE_TEXT) |
|---------------------------|
| Bob said Thank you        |
| Bob said Hello            |
| Bob said Hello World      |
+---------------------------+
```

### web browser経由で利用する方法

サービスはエンドポイントをパブリックに公開する機能を持っていることは先程解説いたしました。そのためサービスがインターネットに公開する Web UI にログインし、Web ブラウザからサービスにリクエストを送信することができます。

DESCRIBE SERVICE コマンドで　サービスが公開するパブリック エンドポイントの URLを見つけます。

```sql
DESCRIBE SERVICE echo_service;
```

本来ならばここでpublic_endpoints列が返ってきて、下記のようなエンドポイントが確認できるのですが、私の環境ではpublic_endpointsが確認できませんでした。こちらはsnowflake社に問い合わせ中です。

```bash
({"echoendpoint":"asdfg-myorg-myacct.snowflakecomputing.app"})
```


**(追記)**
snowvillageのslackにてchatworkのみっつさんより、snowflake内部で動くk8sの仕様であることをおしえていただきました！
active-nodeがすぐに確保出来なかったから、表示されない状態になる可能性があるようです。compute-poolはすぐ起動して処理できるワケではなくactive-nodeを確保して動き始めるので、この場合、active-nodeが確保してからSERVICEが動く事になるので、その段階にならないとendポイントも作られず表示されないとのことです！

また、KDDIのsakatokuさんより、エンドポイントを確認する専用コマンドがあることをおしえていただきました！

```sql
SHOW ENDPOINTS IN SERVICE echo_service;
```
https://docs.snowflake.com/en/sql-reference/sql/show-endpoints
なにか壁に当たったときに皆で一緒に問題を議論できるのはコミュニティのいいところですね〜

私がこちらを叩いたところ以下の表示になっていたので、数分待ってみると無事エンドポイントが確認できるようになりました。
「Endpoints provisioning in progress... check back in a few minutes」

というわけで引き続き「web browser経由で利用する方法」をやっていきます。

ingress_url列に表示されたエンドポイントに「/ui」をつけてbrowserよりアクセスしてみます。

```
https://[ingress_url列に表示されたエンドポイント]/ui
```

アクセスできました！！初回はログインを要求されるので、snowflakeアカウントにろぐいんするときのlogin_nameとpasswordを入れてください。
![](https://storage.googleapis.com/zenn-user-upload/cfbc0bb016b5-20231224.png)

UIが表示されました！
![](https://storage.googleapis.com/zenn-user-upload/2ad5804b6a90-20231224.png)

様々な文字列をinputに入力すると、同期的にoutputに返してくれます...!これは夢が広がりますね...!
![](https://storage.googleapis.com/zenn-user-upload/a7edf5744cf2-20231224.gif)

ここまでがTutorial 1になります。

Tutorial 2と3はJob機能を使用していきますが、2023年12月現在では PrPrですので、またの機会に紹介していきたいと思います。ドキュメントを読むだけでも夢が広がる機能なので、楽しみに待っていたいですね。

https://docs.snowflake.com/en/developer-guide/snowpark-container-services/working-with-jobs

## コンピューティング プールの削除

アクティブなコンピューティングプールノードに対して料金が発生しますので、Tutorial 2, 3をやらない方は不要な料金が発生しないようにコンピューティング プール ノードを削除していきます。

まず、コンピューティング プールで現在実行されているすべてのサービスを停止します。次に、コンピューティング プールを一時停止するか (後で再度使用する場合)、削除します。

コンピューティング プール上のすべてのサービスとジョブを停止します。

```sql
ALTER COMPUTE POOL tutorial_compute_pool STOP ALL;
```

コンピューティング プールを削除します。

```sql
DROP COMPUTE POOL tutorial_compute_pool;
```

イメージ レジストリ (すべてのイメージを削除) や内部ステージ (仕様を削除) をクリーンアップすることもできます。

```sql
DROP IMAGE REPOSITORY tutorial_repository;
DROP STAGE tutorial_stage;
```

## Snowpark Container Serviceのコンテナ費用

snowpark container service の費用は[こちら](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/accounts-orgs-usage-views)にまとまっています。Snowpark Container Serviceのコンテナ費用は「保管コスト」、「コンピューティングプールのコスト」、「データ転送コスト」の3つがかかります。

### ****保管コスト****

Snowpark Container Services を使用する場合、Snowflake ステージの使用量やデータベース テーブル ストレージのコストを含む、Snowflake に関連するストレージ コストが適用されます。

1. **イメージリポジトリのストレージコスト**: イメージリポジトリにはSnowflakeステージを使用します。そのため、Snowflakeステージの利用に関連するコストが発生します。
2. **ログストレージコスト**: ローカルコンテナーログをイベントテーブルに保存する場合、イベントテーブルストレージのコストが適用されます。
3. **ボリュームマウントのコスト**:
    - Snowflakeステージをボリュームとしてマウントする際は、Snowflakeステージの使用によるコストが発生します。
    - コンピューティングプールノードからストレージをボリュームとしてマウントする場合、そのストレージはコンテナー内のローカルストレージとして扱われます。こちらの場合、ローカルストレージのコストはコンピューティングプールノードのコストに含まれるため、追加のコストは発生しません。

### ****コンピューティングプールのコスト****

コンピューティングプールのコストはコンピューティング プール内のノードの数とインスタンス ファミリータイプ(「[CREATE COMPUTE POOL 」](https://docs.snowflake.com/en/sql-reference/sql/create-compute-pool)を参照) によって、消費されるクレジットが決まり、したがって支払うコストが決まります。

[CreditConsumptionTable.pdf](https://www.snowflake.com/legal-files/CreditConsumptionTable.pdf)と[create-compute-pool](https://docs.snowflake.com/en/sql-reference/sql/create-compute-pool)を参照すればコスト見積もりが可能です。

![](https://storage.googleapis.com/zenn-user-upload/066f33ce7aa9-20231222.png)


| Mapping | INSTANCE_FAMILY | vCPU | Memory (GiB) | Storage (GiB) | GPU | GPU Memory (GiB) | Description |
| --- | --- | --- | --- | --- | --- | --- | --- |
| CPU / XS | CPU_X64_XS | 2 | 8 | 250 | 該当なし | 該当なし | Snowpark コンテナで利用可能な最小のインスタンス。コスト削減や入門に最適です。 |
| CPU / S | CPU_X64_S | 4 | 16 | 250 | 該当なし | 該当なし | コストを節約しながら複数のサービス/ジョブをホストするのに最適です。 |
| CPU / M | CPU_X64_M | 8 | 32 | 250 | 該当なし | 該当なし | フルスタック アプリケーションまたは複数のサービスを使用する場合に最適 |
| CPU / L | CPU_X64_L | 32 | 128 | 250 | 該当なし | 該当なし | 異常に大量の CPU、メモリ、ストレージを必要とするアプリケーション向け。 |
| ハイメモリCPU / S | HIGHMEM_X64_S | 8 | 64 | 250 | 該当なし | 該当なし | メモリを大量に使用するアプリケーション向け。 |
| ハイメモリCPU / M | HIGHMEM_X64_M | 32 | 256 | 250 | 該当なし | 該当なし | 単一マシン上で複数のメモリ集中型アプリケーションをホストする場合。 |
| ハイメモリCPU / L | HIGHMEM_X64_L | 128 | 1024 | 250 | 該当なし | 該当なし | 大規模なメモリ内データの処理に利用できる最大の高メモリ マシン。 |
| GPU / S | GPU_NV_S | 8 | 32 | 250 | 1 NVIDIA A10G | 24 | Snowpark Container を開始するために利用できる最小の NVIDIA GPU サイズ。 |
| GPU / M | GPU_NV_M | 48 | 192 | 250 | 4 NVIDIA A10G | 96 | コンピューター ビジョンや LLM/VLM などの集中的な GPU 使用シナリオ向けに最適化 |
| GPU / L | GPU_NV_L | 192 | 2048年 | 250 | 8 NVIDIA H100 | 640 | LLM やクラスタリングなどの特殊かつ高度な GPU ケース向けの最大の GPU インスタンス。 |

コンピューティング プールが IDLE、ACTIVE、STOPPING、RESIZING 状態の場合は料金が発生しますが、STARTING または SUSPENDED 状態の場合は料金が発生しません。（コンピューティングプールのリソース状態は「**[コンピューティング プールのライフサイクル](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/working-with-compute-pool#compute-pool-lifecycle)**」の項目を参照）

## ****データ転送コスト****

データ転送は、Snowflake にデータを移動 (入力) し、Snowflake からデータを移動 (出力) するプロセスです。

**アウトバウンドデータ転送**

- Snowflakeは、サービスやジョブから他のクラウドリージョンやインターネットへのアウトバウンドデータ転送に対して、標準のデータ転送レートを適用します。

**Cross-AZデータ転送**

- Cross-AZデータ転送は、Snowflake内の異なるコンピューティングエンティティ間（例えば、異なるコンピューティングプール間、またはコンピューティングプールとウェアハウス間）でのデータ移動を指します。
- 現在、SnowflakeではCross-AZデータ転送に対しては課金していませんが、将来的に課金を開始する予定です。

要するに、Snowflakeでデータを他のクラウドリージョンやインターネットに送ったり、Snowflake内の異なるコンピューティングプール間にデータを移動させたりすると、コストがかかります。

## Snowpark Container Serviceが作るデータエンジニアリングの未来
Snowpark Container Serviceにより、データエンジニアリングの未来が一層豊かなものとなることは間違いありません。このサービスは、従来のデータ基盤開発の枠を超えて、データアプリケーションの開発領域を拡大しています。データエンジニアはこれまで、データ基盤の構築と維持に重点を置いてきましたが、Snowpark Container Serviceの登場により、私達の役割はより多岐にわたるものとなります。具体的には、Snowflakeをバックエンドとして利用することにより、データエンジニアはアプリケーション開発にも深く関与できるようになります。これには、Snowflake上でのエンドポイントの作成やAPI定義書の作成などが含まれるでしょう。また、Snowpark CortexのようなLLM（Language Learning Models）の実装にも携わることができるようになり、データエンジニアの担当範囲はこれまで以上に広がります。

データエンジニアが単にデータを管理し分析するだけでなく、データを活用したアプリケーションの開発においても主要な役割を担うようになるような気がしています。Snowpark Container Serviceは、データを活かしてさらなるアプリケーション開発の領域へ押し上げる可能性を秘めているのかもしれません。

## 終わりに
snowflakeはどんどん進化していきますね。我々データエンジニアもsnowflakeの進化に遅れを取らないように、知識のアップデートを進める必要があると感じました。SPCSはsnowflakeをアプリケーション開発に利用している弊社にとっては大変ありがたい機能です。早く東京リージョンでのGAが待ち遠しいですね。今後、使い倒していこうと思います。
