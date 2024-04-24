---
title: "snowflakeでslack通知を実現する3つの方法"
emoji: "❄️"
type: "tech"
topics: [slack, snowflake, dataengineering, bigdata]
published: true
---

## はじめに
データエンジニアの是枝です。
皆さんはsnowflakeから自分の使っているchatツールに通知したいと思うことはないでしょうか。snowflakeで起こったエラーに素早く気づくためには、通知の仕組みを実装するのが監視の基本ですよね。今回はsnowflakeからslack通知の実現方法をいくつかご紹介したいと思います。

## 方法1：EXTERNAL NETWORK ACCESSによる外部アクセス統合を利用する

EXTERNAL NETWORK ACCESSは外部ネットワークにセキュアにアクセスできる方法として知られています。UDFやストアドプロシージャで使用する事ができ、簡単に外部アクセス統合が実現可能です。こちらを利用してslackにリクエストを飛ばして通知を実現します。理論上、APIさえ開放されていれば、いかなるチャットツールにも通知を出すことができる手法となります。流れは以下です。

**1. slackトークンの取得**

まずは https://api.slack.com/apps/ にログインして 「Create New App」をクリックし、必要事項を入力していきます。your app’s scopes and settings.の部分では、`From scratch` を選択します。
![](https://storage.googleapis.com/zenn-user-upload/3c6b29db8e89-20240424.png)

appのBasic Informationが見れるようになるので、Add features and functionally > Permissions と進みます。
![](https://storage.googleapis.com/zenn-user-upload/eef6423783cd-20240424.png)

少しスクロールすると `Scopes` という部分が見えるようになるので、こちらから `Bot Token Scopes` を選び、必要な権限をアサインしていきます。もしUserとして投稿したい場合は、`User Token Scopes`を選びます。
![](https://storage.googleapis.com/zenn-user-upload/f771308eec05-20240424.png)

ちなみに私の設定は下記です。自社の要件に合わせてカスタマイズしてください。
![](https://storage.googleapis.com/zenn-user-upload/ba235886b544-20240424.png)

完了後、Workspaceにインストールしていきます。インストールが完了するとトークンをコピーできるようになります。
![](https://storage.googleapis.com/zenn-user-upload/e170f08d5de8-20240424.png)
![](https://storage.googleapis.com/zenn-user-upload/5c3153fe46b4-20240424.png)

**2. EXTERNAL ACCESS INTEGRATIONを行う**
処理の流れとしては、はじめにnetwork ruleを定義して、ポリシーが `slack.com` というホストに対して適用されるようにします。次に `slack_token` という名前のシークレットを作成してsnowflake内部で秘密情報を管理します。network roleと登録したシークレットを使って`XTERNAL ACCESS INTEGRATION`コマンドにてslackとsnowflakeの外部アクセス統合を作ります。

```sql
CREATE OR REPLACE NETWORK RULE slack_rule
  MODE = EGRESS
  TYPE = HOST_PORT
  VALUE_LIST = ('slack.com');

CREATE OR REPLACE SECRET slack_token
  TYPE = GENERIC_STRING
  SECRET_STRING = '[slack-token]';

CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION slack_apis_access
  ALLOWED_NETWORK_RULES = (slack_rule)
  ALLOWED_AUTHENTICATION_SECRETS = (slack_token)
  ENABLED = true;
```

**3. slackにポストする**
あとは、slackにポストするような処理をUDFやストアドプロシージャ内に実装すれば完了です。
UDFでの実現例は以下のとおりです。こちらでは、Slack の chat.postMessage API を呼び出してメッセージを送信しています。API キー（トークン）をヘッダに設定し、POST リクエストで Slack にデータを送信しています。

```sql
CREATE OR REPLACE FUNCTION post_to_slack(channel varchar ,message varchar)
RETURNS variant
LANGUAGE python
RUNTIME_VERSION = 3.8
HANDLER = 'main'
EXTERNAL_ACCESS_INTEGRATIONS = (slack_apis_access)
PACKAGES = ('requests')
SECRETS = ('cred' = slack_token)
AS
$$
import _snowflake
import requests
import json
def main(p_channel ,p_message):
    api_key = _snowflake.get_generic_secret_string('cred')
    url = 'https://slack.com/api/chat.postMessage'
    v_headers = {"Authorization": "Bearer "+ api_key}
    v_data_json = {
    'channel': p_channel
    ,'text': p_message
    }
    response = requests.post(url ,headers=v_headers ,json = v_data_json)
    return response.json()
$$;
```
こちらで試しに実行できます。
```sql
SELECT
'[channel ID]' as channel
,'TESTです' as message
,post_to_slack(channel ,message) as slack_notify
```

ちなみにchannel IDは、チャンネル名をクリックして詳細を開いた後に現れるポップアップの一番下で確認できます。
![](https://storage.googleapis.com/zenn-user-upload/e218bbca8cd7-20240424.png)


下記のようなイメージで、`slack_notification` という関数を一つ定義しておけば、動的なslack通知をストアドプロシージャ内で実現できます。こちらのコードはエラー通知に利用しています。
```python
def slack_notification(session, channel_id, message):
    notification_query = f"""
    SELECT
    '{channel_id}' as channel,
    '{message}' as message,
    post_to_slack(channel, message) as slack_notify;
    """

    session.sql(notification_query).collect()

def create_database(session, database_name):
    try:
        session.sql(f"CREATE DATABASE IF NOT EXISTS \"{database_name}\"").collect()
        return f"データベース '{database_name}' が正常に作成されました。"
    except Exception as e:
        error_message = f"データベース '{database_name}' の作成中にエラーが発生しました。エラー内容: {e}"

        # エラー発生時にSlack通知
        slack_notification(session, 'error-log-channel-id', error_message)
        return error_message

```

こちらを実装するにあたり[snowflakeの高田さんの記事](https://zenn.dev/tf_takada/articles/d5d2c6c03b39a8)の内容を参考にさせていただきました。

## 方法2：NOTIFICATION INTEGRATIONを使ってslackチャンネルメールアドレスに通知
NOTIFICATION INTEGRATIONにslackチャンネルメールアドレスを指定してしまう最も簡単かつ軽量なやり方です。ネットワークに関するあれこれ難しいことを考えたくないよ...といった方におすすめです。

snowflakeには `SYSTEM$SEND_EMAIL`というメール通知を行うシステム関数が用意されていますのでこちらを使っていきます。
https://docs.snowflake.com/ja/sql-reference/stored-procedures/system_send_email

**使用例**
```sql
CALL SYSTEM$SEND_EMAIL(
'my_email_list',
'メールアドレス',
'件名',
'本文')
```

処理の流れは、slackチャンネルメールアドレスをNOTIFICATION INTEGRATIONのメールリストに登録してSYSTEM$SEND_EMAILでslackチャンネルメールアドレス宛に送信してしまう方法です。手順は以下のとおりです。


**1. slackチャンネルのメールアドレスを取得する**
channelの詳細のインテグレーションタブを押すと、下の方に「このチャンネルにメールを送信する」と出ており、channelのメールアドレスを発行することができます。（ただし有料プランのみ）
![](https://storage.googleapis.com/zenn-user-upload/8aa22eba9063-20240424.png)

**2. NOTIFICATION INTEGRATIONでメールリストを作成する**
```sql
CREATE NOTIFICATION INTEGRATION my_email_list
    TYPE=EMAIL
    ENABLED=TRUE
  ALLOWED_RECIPIENTS=('x-asdfghjklasdfghj@hogehoge.slack.com');
```

**3. 設定したメールアドレスでSYSTEM$SEND_EMAILを呼び出す**
あとは、SYSTEM$SEND_EMAILをcallすればメール送信できます。
```sql
CALL SYSTEM$SEND_EMAIL(
'my_email_list',
'x-asdfghjklasdfghj@hogehoge.slack.com',
'snowflake通知テスト',
'koreeda_testです')
```
![](https://storage.googleapis.com/zenn-user-upload/8ba413f4300c-20240424.png)


以下のように動的な文章も生成可能です。
```sql
SET my_dynamic_body = (SELECT '本日の日付は ' || CURRENT_DATE() || ' です。データベースシステムは正常に稼働しています。');

SET additional_message = '何か問題が発生した場合は、すぐにシステム管理者に連絡してください。';

SET full_message = $my_dynamic_body || '\n\n' || $additional_message;

CALL SYSTEM$SEND_EMAIL(
'my_email_list',
'x-asdfghjklasdfghj@hogehoge.slack.com',
'snowflake通知テスト',
$full_message);
```

もちろんストアドプロシージャ内でも使用可能です。以下のようにメール送信専用のストアドプロシージャを用意して、既存のストアドプロシージャからcallすれば通知機能が簡単に実装できます。

```sql
CREATE OR REPLACE PROCEDURE send_dynamic_email()
RETURNS STRING
LANGUAGE PYTHON
RUNTIME_VERSION = '3.8'
HANDLER = 'send_email_handler'
PACKAGES = ('snowflake-snowpark-python==0.7.0')
AS
$$
def send_email_handler(session):
    session.sql("CALL SYSTEM$SEND_EMAIL('my_email_list', 'x-asdfghjklasdfghj@hogehoge.slack.com', 'snowflake通知テスト', 'hogehoge')")
    return "メール送信が成功しました。"
$$;

/* 使用コマンド
call send_dynamic_email();
/*
```

注意点としては、**ALLOWED_RECIPIENTSに指定できるのはメール検証されているユーザーのメールだけ** という点です。メール検証のやり方は、My profileをクリックして、
![](https://storage.googleapis.com/zenn-user-upload/5a695c889496-20240424.png)

上の方にResend email valification的なものが出ているので再送します。
![](https://storage.googleapis.com/zenn-user-upload/1054d68b89bb-20240424.png)


こちらを実装するにあたり[select社の記事](https://select.dev/posts/snowflake-alerts#how-to-send-notifications-to-slack-from-snowflake)の内容を参考にさせていただきました。


## 方法3：クラウドメッセージングを利用する
最後は、AWS SNS トピックなどのクラウドメッセージングを利用する方法です。おそらく「snowflake タスク 通知」などでググると真っ先に公式ドキュメントが出てくるのでこのやり方を使っている人は多いのではないかと思います。
https://docs.snowflake.com/ja/user-guide/tasks-errors

以下のようなクラウドメッセージングサービスを利用できます。注意点は、クロスクラウドサポートされていないので、Snowflakeアカウントがホストされているクラウドプラットフォームによって提供されるという点です。
・Amazon Simple Notification Service（SNS）
・Microsoft Azure Event Grid
・Google Pub/Sub

エラー通知を受信するトピックを作成し、そのトピックの通知統合を設定します。

今回はAWS SNSに絞って解説していきます。流れとしてはAmazon SNS トピックの作成→IAM ポリシーとロールを作成→NOTIFICATION INTEGRATIONでtopic_arnとiam_role_arnを指定するイメージです。

![](https://storage.googleapis.com/zenn-user-upload/a4e9a2de0146-20240425.png)


**NOTIFICATION INTEGRATION**のコマンドはこちらです。
```sql
CREATE NOTIFICATION INTEGRATION <integration_name>
  ENABLED = true
  TYPE = QUEUE
  NOTIFICATION_PROVIDER = AWS_SNS
  DIRECTION = OUTBOUND
  AWS_SNS_TOPIC_ARN = '<topic_arn>'
  AWS_SNS_ROLE_ARN = '<iam_role_arn>'
```

Data SuperHeroのあれさんの記事がわかりやすくまとまっているので、是非参考にしてください！
https://zenn.dev/allllllllez/articles/4ee1ad9acf6a80


## それぞれの方法のメリット・デメリット
それぞれのメリット・デメリットを書き出してみると以下のようなイメージです。

| 通知方法                              | メリット                                              | デメリット                                              |
|-----------------------------------|---------------------------------------------------|----------------------------------------------------|
| 1. EXTERNAL ACCESS INTEGRATION利用法 | Snowflake内で完結し、セキュリティを確保しつつAPI経由での通知が可能。カスタマイズ性が高い。 | 外部アクセスの設定にはネットワークのルールとセキュリティポリシーを理解し管理する必要がある。 |
| 2. Slackメアド通知法                   | 簡単で速い設定が可能。Slackの既存のemail-to-channel機能を利用して手軽に通知。    | Slackの有料プランが必要であり、通知のカスタマイズ性に制限がある（基本的にはテキストメールの形式）。 |
| 3. クラウドメッセージング法               | プラットフォームに依存しない広範なクラウドサービス（AWS SNSなど）を利用することで高い拡張性と信頼性を確保。 | クラウドサービスの設定と管理が必要であり、コストが増加する場合がある。Snowflakeの実行環境と同じクラウドプロバイダを選ぶ必要がある。 |


社内のネットワークポリシーが特に厳格でなければ、Snowflake内で完結しセキュリティも確保できる外部アクセス統合（方法1）が適しています。一方で、最も簡単な方法で通知機能を実装したい場合は、Slackのemail-to-channel機能を活用する方法（方法2）がお勧めです。また、AWSなどのクラウドシステムがすでに整備されており、その保守も問題ない場合は、クラウドメッセージング（方法3）を利用するのが最適です。個人的にはsnowflake内で完結できて通知スタイルのカスタマイズも可能な **`1. EXTERNAL ACCESS INTEGRATION利用法`** がおすすめで、弊社でも取り入れています。

## 終わりに
いかがだったでしょうか。色々なやり方があると思うので迷うと思いますが、自社にあった通知方法で実装していただければと思います！
