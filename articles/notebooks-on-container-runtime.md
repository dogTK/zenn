---
title: "真のSnowflakeデータサイエンス時代の幕開け Notebooks on Container Runtimeとは"
emoji: "❄️"
type: "tech"
topics:
  - "snowflake"
  - "notebooks"
  - "datascience"
  - "container"
  - "machinelearning"
published: false
---

## 記事の結論
- Notebooks on Container Runtimeを使うとパッケージの幅が広がるよ
- Warehouse駆動より安上がりで済むよ

## はじめに
データエンジニアの是枝です。

2024年10月2日にNotebooks on Container Runtime が Previewになりました！

https://docs.snowflake.com/en/release-notes/2024/other/2024-10-02-notebooks-on-spcs

こちらが出た際、私はSnowflakeのデータサイエンスの新時代が幕開けしたと革新しました。使い心地もGoogle Colaboratoryに近づいてきてると思います。これまでのNotebooksと何が違うのか？と疑問に思う方もいると思うので、Notebooks on Container Runtimeの概要と具体的な実装コード付きで解説していきたいと思います。

*Snowpark Container Servicesがなにかわからないという方は、手前味噌となりますが、私がSPCS入門向けの記事を書いておりますのでこちらを参考にしてください。
https://zenn.dev/dataheroes/articles/3437ca8c0eddfc

## Notebooks on Container Runtimeとは？
Container Runtime は NotebooksをSnowpark Container Servicesで動かす機能です。これによりSnowflake内で高度なデータサイエンスと機械学習のワークロードのためのソフトウェアとハ​​ードウェアのオプションを提供します。Notebooks on Container Runtimeを使う具体的なメリットを記載します。　Snowpark Container Servicesのように **サービスを作成したりDocker imageを用意したりする必要はなく、Compute Poolの準備さえ行えば使うことができます。**

### 1. pipインストールの公式サポート
Snowflakeが公式でサポートするPython Packageに限りがありサードパーティのパッケージ使用のためには、[Snowflakeステージを介したパッケージのインポート](https://docs.snowflake.com/ja/developer-guide/udf/python/udf-python-packages#importing-packages-through-a-snowflake-stage)といった手間がかかる方法を行う必要があり、利便性が落ちていました。

Notebooks on Container Runtimeでは、パブリックリポジトリから追加のパッケージをインストールする際に **pipコマンド** を使用することができます。これにより利便性を保ったままsnowflakeが公式サポートするパッケージ以外のものも使用することが可能になりました。



一つだけ注意点があって、Snowflakeのリソースにクエリしたりするコードを実行するなどの場合は、ウェアハウスも合わせてかかります。SQLセルでSQLを使用し、Pythonセルで実行されるSnowparkによるプッシュダウンクエリを使用する場合は、Warehouseを動かす必要があるということです。

![](/images/notebooks-on-container-runtime/notebook-compute-diagram.png)
*https://docs.snowflake.com/en/user-guide/ui-snowsight/notebooks-on-spcs#cost-billing-considerations より引用*



## Notebooks on Container Runtimeを使う準備

### コンピュートプールの作成
```sql
USE ROLE accountadmin;

CREATE COMPUTE POOL datascience_compute_pool
  MIN_NODES = 1
  MAX_NODES = 1
  INSTANCE_FAMILY = CPU_X64_XS;
```
まずはNotebooks on Container Runtimeに使うcompute_poolを最小単位（最小コスト）で作ります。ちなみにXSのノード数1だと8GiBメモリだけで並列処理ができないので、やりたい解析に応じてINSTANCE_FAMILYサイズを上げたりノード数を調整してください。

参考：https://docs.snowflake.com/en/sql-reference/sql/create-compute-pool#required-parameters

### ロールを作成、権限付与
```sql
CREATE ROLE DATASCIENTIST;

GRANT CREATE NOTEBOOK ON SCHEMA [SCHEMA_NAME] TO ROLE DATASCIENTIST;
GRANT CREATE SERVICE ON SCHEMA [SCHEMA_NAME] TO ROLE DATASCIENTIST;
GRANT USAGE ON COMPUTE POOL datascience_compute_pool TO ROLE DATASCIENTIST;
```
ACCOUNTADMIN、ORGADMIN、または SECURITYADMIN のロールを持つユーザーは、Container Runtimeでノートブックを直接作成したり所有したりすることはできず、実行もできないという制約があります。そのため、Container Runtime用のロールを作成し、必要な権限をGrantしていきます。[SCHEMA_NAME]の部分はご自身の環境に合わせて適宜変更してください。

### ネットワークルール、EXTERNAL ACCESS INTEGRATIONの作成
```sql
create network rule allow_all_rule
  TYPE = 'HOST_PORT'
  MODE= 'EGRESS'
  VALUE_LIST = ('0.0.0.0:443','0.0.0.0:80');

CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION allow_all_integration
  ALLOWED_NETWORK_RULES = (allow_all_rule)
  ENABLED = true;

CREATE OR REPLACE NETWORK RULE pypi_network_rule
  MODE = EGRESS
  TYPE = HOST_PORT
  VALUE_LIST = ('pypi.org', 'pypi.python.org', 'pythonhosted.org',  'files.pythonhosted.org');

CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION pypi_access_integration
  ALLOWED_NETWORK_RULES = (pypi_network_rule)
  ENABLED = true;

GRANT USAGE ON INTEGRATION allow_all_integration TO ROLE DATASCIENTIST;
GRANT USAGE ON INTEGRATION pypi_access_integration TO ROLE DATASCIENTIST;
```
コンテナが外部通信するために、ネットワークルール、EXTERNAL ACCESS INTEGRATIONを作成していきます。

これでContainer Runtimeを使う準備は完了です。

## Notebooks on Container Runtimeを使ってみる
それでは実際にcontainer runtimeを使ってみましょう。先ほど作成したDB, スキーマを選び、Python Environmentで「Run on container」を選択します。
![](/images/notebooks-on-container-runtime/notebooks_select_container.png)

![](/images/notebooks-on-container-runtime/Runtime.png)

![](/images/notebooks-on-container-runtime/compute_pool.png)

ちなみにACCOUNTADMINのまま使おうとすると、専用ロールに切り替えるようエラーがでるので、今回でいうとDATASCIENCEロールにきちんと切り替えを行ってから実行してください。
![](/images/notebooks-on-container-runtime/not_use_accountadmin.png)



![](/images/notebooks-on-container-runtime/external_access.png)

### 内部ステージにファイルアップロード


### Notebooks on Container Runtimeからファイルを呼び出す
Notebooksを新規作成したら設定画面が出てくるので、「Run on container」を選びます。
RuntimeはCPUとGPUを選ぶことができます。
そして、先程作成したコンピュートプールを選択したらCreateをクリックしましょう。



## clean up
最後にコストがかからないようにコンピュートプールを削除しましょう。
```sql
ALTER COMPUTE POOL datascience_compute_pool STOP ALL;
DROP COMPUTE POOL datascience_compute_pool;
```



## まとめ
いかがだったでしょうか。Notebooks on Container Runtimeを使うとパッケージの幅が広がる上に、コンピュートリソース駆動より安上がりで済ます事ができます。
