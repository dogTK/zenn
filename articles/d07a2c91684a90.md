---
title: "Data for Breakfast Osakaに参加した感想"
emoji: "⛄"
type: "tech"
topics:
  - "データ分析"
  - "データベース"
  - "snowflake"
  - "データ基盤"
  - "データクラウド"
published: true
published_at: "2023-06-08 23:45"
---

データエンジニアの是枝です。

2023年6月8日、snowflake社主催のData for Breakfast Osakaに参加してきました！

データクラウドへの出発点
大阪 | ラ・フェット ひらまつ | 6月8日(木)
定員：70名
https://www.snowflake.com/event/data-for-breakfast-osaka/?lang=ja


私自身、関西に住んでいるため、関東で開催される物理イベントに参加できずにもどかしい思いをすることが多かったのですが、この度Data for Breakfastが大阪で開催されるとのことで、早速現地参加してきました！

## 会場の雰囲気
前回snowflakeのイベントに参加した時はsnowdayJapanだったのですが、その時にsnowflakeのイベントはとにかく豪華...!思いました。今回のData for Breakfastは様々の都市を回っており、大阪の前は東京でも行われています。東京のイベントレポートをいくつか見てみてから、「やはり今回のイベントも豪華そうだ...!」と期待値が高まっておりました。

会場が格調高くて、ジャケットを羽織ってきて本当に良かったと思いました笑
ちなみに簡単な軽食が無料で食べられます。Snowflakeさん、ホスピタリティがすごい....
![](https://storage.googleapis.com/zenn-user-upload/1c145dad4c0b-20230608.jpeg)
![](https://storage.googleapis.com/zenn-user-upload/e3bc87719be1-20230608.jpeg)
![](https://storage.googleapis.com/zenn-user-upload/ed823bbff9af-20230608.jpeg)
![](https://storage.googleapis.com/zenn-user-upload/1e725edaa571-20230608.jpeg)
![](https://storage.googleapis.com/zenn-user-upload/4f5c2689c5b4-20230608.jpeg)



## 実際の講演
**snowflake社からオープニング & ウェルカム**にてSnowflakeチームから開会のご挨拶とSnowflakeデータクラウドの概要について紹介がありました。
**データクラウドの紹介**タサイロの解消と適切なガバナンスのもとでのほぼすべての保有データへのアクセスを可能にする方法が解説されていました。

snowflakeはいつの間にか皆さんのコストを20%削減してくれていたようです...!
![](https://storage.googleapis.com/zenn-user-upload/b98949743946-20230608.jpeg)


いろんなデータ基盤と簡単にコラボレーションやデータ共有ができるのはsnowflakeの強みですよね~
![](https://storage.googleapis.com/zenn-user-upload/18abc7b6f71e-20230608.jpeg)

snowflakeはデータレイク層を内部に用意してrawデータをまるっと取り込むアーキテクチャを推奨しているようです。内部にデータがある方が参照部分がボトルネックにならずに済むからありがたい。
![](https://storage.googleapis.com/zenn-user-upload/2c1224b0d997-20230608.jpeg)


## 住友ゴムのデータドリブン文化醸成に向けた360°推進
住友ゴム工業株式会社
経営企画部 デジタルイノベーション推進グループ
兼 製造IoT推進室　課長
金子 秀一　さん

snowflakeを用いたデータドリブン文化醸成に向けた取り組みについて紹介されていました。

![](https://storage.googleapis.com/zenn-user-upload/c26faab976fd-20230608.jpeg)

特に注目したいのが、データドリブンの文化を醸成する方法を表したこちらの図。一人二人でもデータを使ってくれると成功事例が溜まって、他の人に波及していきますよね。そのようなサイクルを起こせればデータドリブン文化が醸成していくのだろうなと想像できました。また、「基盤を整備するのが先か、データ利用を進めるのが先かは」大変悩ましい問題ですが、鶏が先か卵が先か問題でどちらが良いとはいえないみたい。
![](https://storage.googleapis.com/zenn-user-upload/933dbf512ff6-20230608.jpeg)


tableauからクエリが返ってくるのが遅すぎてデータが使われなくなっていく課題感を解決するためにsnowflakeを導入したそうです。96%削減はすごい..(画像が何故か黄色くてすいません)
![](https://storage.googleapis.com/zenn-user-upload/70350ae68e9d-20230608.jpeg)


弊社でもデータマネジメントのガイドライン制定を進めなきゃなーと思わされたスライド。どういう構成にすればよいか大変参考になる。
![](https://storage.googleapis.com/zenn-user-upload/d8a37a99c827-20230608.jpeg)

データ活用を推進する熱量がすごかったです。社内勉強会も弊社で企画しなきゃなーと思わされました。
![](https://storage.googleapis.com/zenn-user-upload/8ca32b388ccd-20230608.jpeg)




## マーケットプレイスパネルセッション
株式会社truestar 代表取締役社長
truestar hd 株式会社 取締役

藤 俊久仁　さん

株式会社HR Force CDO
法人番号株式会社　代表取締役

吉田 裕宣　さん

Snowflakeマーケットプレイスでは数百のプロバイダから提供されるライブデータ、データサービス、アプリケーションデータを活用できます。実際に日本でもマーケットプレイスを活用し、より深いインサイトで高度なアナリティクスを実現している例が紹介されていました。
![](https://storage.googleapis.com/zenn-user-upload/076e6a759720-20230608.jpeg)
![](https://storage.googleapis.com/zenn-user-upload/ca0e0bc921c8-20230608.jpeg)

snowflakeのアカウントが共有先にもあれば、データをレプリケーションせずにクエリだけを投げれるのは画期的なアーキテクチャだなと感じます。データ非公開にしたければアクセス制御すれば簡単にデータ共有もやめることができます。
![](https://storage.googleapis.com/zenn-user-upload/2a37e3435547-20230608.jpeg)




何より、法人番号株式会社のマーケットプレイスに共有しているPrepper Open Data Bank - Japanese Corporate Data - Examples(PBOD)を用いたデモはまじでヤバかったです。笑

Prepper Open Data BankのJapanese Corporate Dataには、経済産業省のgBizINFOの公開データを元に、truestar社が加工した日本国内の法人情報が入っています。共有する法人情報には全て法人番号が含まれており、法人番号をキーに任意の企業の情報を簡単に抽出することが可能です。

以下の情報が法人番号をキーに接続できるなんて...素晴らしいデータだ！

Sample fields
- CORPORATE_NUMBER | 法人番号
- NAME | 法人名
- KANA | 法人名ふりがな
- NAME_EN | 法人名英語
- POSTAL_CODE | 郵便番号
- LOCATION | 本社所在地
- STATUS | ステータス
- CLOSE_DATE | 登記記録の閉鎖等年月日
- CLOSE_CAUSE | 登記記録の閉鎖等の事由
- REPRESENTATIVE_POSITION | 法人代表者役職
- REPRESENTATIVE_NAME | 法人代表者名
- CAPITAL_STOCK | 資本金
- EMPLOYEE_NUMBER | 従業員数
- COMPANY_SIZE_MALE | 企業規模詳細(男性)
- COMPANY_SIZE_FEMALE | 企業規模詳細(女性)
- BUSINESS_ITEMS | 営業品目リスト
- BUSINESS_SUMMARY | 事業概要
- COMPANY_URL | 企業ホームページ
- DATE_OF_ESTABLISHMENT | 設立年月日
- FOUNDING_YEAR | 創業年
- UPDATE_DATE | 最終更新日
- PODB_UPDATE_DATE | PODB更新日
- PODB_UPDATE_DETAIL | PODB更新内容




snowflakeとsalesforceをtroccoのフリープランで繋いで、常に最新の企業情報をsalesforceにインポートできる仕組みをさらっと実装されていました。（え、そんなことできるん）
法人番号はCRMや顧客プロダクトデータとの連携をする上で重要なキーとなります。私は過去にプロダクトDBに法人番号を流し込むバッチ処理を何度も作成した経験があり、この作業の手間は重々理解をしておりました。データを最新に更新できる仕組みが無料で実現できるのは大変ありがたい世の中だな~(当時の私に教えてあげたいくらいです)




![](https://storage.googleapis.com/zenn-user-upload/ee844acc6f63-20230608.jpeg)

弊社のデータもしっかり入っておりました！
![](https://storage.googleapis.com/zenn-user-upload/0a07f599ead8-20230608.png)



## 終わりに
やはりsnowflakeさんの開催するイベントは内容もホスピタリティも充実してて毎回満足感が高いですね。同じテーブルだった方々と名刺交換もできて交流もできました。また機会があればぜひ参加したいです。



twitterでsnowflakeのことをよく呟いているのでよかったらフォローしてください！
https://twitter.com/cs_dev_engineer/status/1666632716157157378