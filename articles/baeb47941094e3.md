---
title: "SnowflakeでParquet形式の半構造化データを効率的にロードする方法: PARSE_JSONを使って中間テーブルを省略"
emoji: "❄️"
type: "tech"
topics:
  - "データ分析"
  - "データベース"
  - "snowflake"
  - "データアナリスト"
  - "データエンジニア"
published: true
published_at: "2023-04-20 20:14"
---

![](https://storage.googleapis.com/zenn-user-upload/ef9c2ed018be-20230420.png)

こんにちは、データエンジニアの是枝です。

最近、42億レコードをRDSからSnowflakeに入れる仕事を任されています。
https://twitter.com/cs_dev_engineer/status/1646500199593033728

S3にアップロードしたparquet形式のDBスナップショットをSnowflakeに入れる手法を取ろうと思いましたが一筋縄では行かなかったので、備忘録としてまとめます。

# TL;DR
具体的にはSnowflake の PARSE_JSON 関数を利用して、中間テーブルを作成せずに、半構造化データとして保存したParquet 形式のファイルを効率的にテーブルに変換する方法の紹介です。この方法により、SQL の実行回数が削減され、データ変換の手間も軽減されます。これを機に、半構造化データを扱う際の効率とパフォーマンスを向上させることができると思います。

# 課題

DBスナップショットはParquet形式を利用することが一般的です。
これをアップロードするには、一時テーブルを作成した後に、Createでテーブルで型指定する２つのSQLで実装する必要があります。

```sql
# 一時テーブルを作成
CREATE TEMPORARY TABLE temp_my_data AS
  SELECT *
  FROM @EXTERNAL_STAGE;
  
# テーブルを作成
CREATE TABLE my_data AS
SELECT
  $1:id::integer as ID,
  $1:value::VARCHAR as VALUE,
  $1:created_at::TIMESTAMP_NTZ(9) as CREATED_AT
FROM temp_my_data;
```

しかしながらこの手法だと、SQL を複数回実行する必要があるため、若干手間がかかります。なんとか中間テーブルを介さずに一気にテーブルをロードしたい感があります。


# Parquet形式の半構造化データをSnowflakeにロードする際に、PARSE_JSONが使える

Snowflake では、PARSE_JSON 関数を使用して、JSON 文字列を VARIANT 型に変換することができます。VARIANT 型は、半構造化データ（JSON、Avro、Parquet など）を効率的に表現し、格納するために使用されます。

PARSE_JSON 関数は、半構造化データを操作する際に非常に便利で、データの変換や抽出に役立ちます。キーに対応する値を抽出した後は、さらに適切なデータ型にキャストすることができます。

**PARSE_JSON**
https://docs.snowflake.com/ja/sql-reference/functions/parse_json

先程の中間テーブル作成手法ではSQLを２つ走らせないと行けなかったですが、PARSE_JSON 関数を利用すると以下のようにまとめる事ができます。

```sql
CREATE OR REPLACE TABLE my_data_2 AS
  SELECT
  $1:id::integer as ID,
  $1:value::VARCHAR as VALUE,
  $1:created_at::TIMESTAMP_NTZ(9) as CREATED_AT
  FROM
    (SELECT
      PARSE_JSON($1) as data
    FROM
      @EXTERNAL_STAGE);
```

こちらで中間テーブルを介さずともほしいテーブルを一つのSQLで得ることができます。

# 終わりに
いかがだったでしょうか。PARSE_JSON 関数の利用により、中間テーブルを作成せずに、Parquet 形式の半構造化データを効率的に Snowflake のテーブルに変換することができました。これにより、SQL の実行回数を削減してデータ変換の手間も軽減されます。この方法を活用して、半構造化データを扱う際の効率とパフォーマンスをさらに向上させることが期待できます。

他のやり方をご存知の方がいらっしゃいましたらぜひコメント欄にて教えて下さい。