---
title: "RstudioでGitHub Copilotが使えるようになったので試してみる"
emoji: "🧬"
type: "tech"
topics:
  - "r"
  - "rstudio"
  - "バイオインフォマティクス"
  - "githubcopilot"
  - "generativeai"
published: true
published_at: "2023-07-16 18:02"
---

データエンジニア兼バイオインフォマティシャンの是枝です。

最近の生成系AIの勢いはとどまることを知らないですね。今日始めてOpen AI Code Interpreterを試してみましたが簡単なグラフならデータを与えるとサクッと作ってくれるのでデータアナリストには大変ありがたい状況になっているのでは無いでしょうか。

しかしながら、学術系の解析をするとうまくグラフ生成してもらえなくて少しモヤッとした印象を受けました。私がまだ上手く使いこなせない可能性はありますが、複雑なデータ分析をするにはプロンプトをかなり工夫する必要はある印象です。やはり自分でコードを書く作業はまだ必要そうだなと感じた次第です。そんなときに役立つのがGitHub Copilotですね。説明するまでも無いかもしれませんがGitHub Copilotはユーザーが書いたコードに基づいて次に書くべきコードを予測してくれるAIベースのコード補完ツールです。

![](https://storage.googleapis.com/zenn-user-upload/fe3485f41beb-20230716.webp)
https://github.com/features/preview/copilot-x

そんなGitHub Copilotですが先日、とうとうRstudio上でも使えるようになったみたいです。
https://twitter.com/u_ribo/status/1672096015565082624
Dairy Builds版のみらしいですが、これはRを使って統計・データ分析をする方には嬉しいのでは無いでしょうか。私みたいなバイオインフォマティクスをする人間にとっても朗報です。

本記事ではRstudioでGitHub Copilotを使ってみる方法を書いていきます。

:::message
github copilotは有料機能なので、試す場合はこちらより登録をお願いします。($10)
https://github.com/github-copilot/signup
:::


## RstudioのDairy Builds版のインストール
RstudioでGitHub Copilotを利用できるのは2023年7月16日現在でDairy Builds版のみです。こちらよりアクセスして自身のOSのインストーラーをダウンロードし、インストールを完了させてください。
https://dailies.rstudio.com/!

[](https://storage.googleapis.com/zenn-user-upload/1af1593e3bd7-20230716.png)

## RstudioからGitHub Copilotへ認証
Tool>Global optionsと進みます。
![](https://storage.googleapis.com/zenn-user-upload/f54f46c9edd7-20230716.png)

左サイドバーから「Copilot」を選び、「Enable GitHub Copilot」にチェックを入れます。
次にSign Inをクリックし、出てきたリンク先でVerification codeを入力してください
![](https://storage.googleapis.com/zenn-user-upload/a2e66361b87c-20230716.png)

![](https://storage.googleapis.com/zenn-user-upload/9109dbdbafe4-20230716.png)

Authorize Github Copilot Pluginをクリックし認証を完了させれば設定は完了です。
![](https://storage.googleapis.com/zenn-user-upload/f99137855dee-20230716.png)


## RstudioでGitHub Copilotを使ってみる
実際にRstodioでGitHub Copilotを使って見ます。
以下のように書こうとするコードが灰色で予測され、タブを押すとコードを受け入れる事ができます。
![](https://storage.googleapis.com/zenn-user-upload/4a6f65b8d91a-20230716.gif)


## 終わりに
いかがだったでしょうか。実際に研究で使ってみるとコードの予測を出してくれるのは大変助かります。あくまでこちらはまだDaily build版であり今後の開発によっては使えなくなる可能性があることをご留意ください。