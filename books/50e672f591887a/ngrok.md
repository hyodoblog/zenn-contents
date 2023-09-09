---
title: "Ngrok を使ったローカル開発環境の構築"
---

こちらのセクションでは、Ngrok を使ってローカル開発環境を構築する方法を紹介します。

過去に書いた記事があるのでこちらを参照するか、以下の手順に従ってください。

https://zenn.dev/hyodoblog/articles/c1b2d6f6135af7

## Ngrok のインストール（Mac)

⚠Mac のみの紹介となりますのでご了承ください。

1. https://ngrok.com/ にアクセスし会員登録する
2. [ダッシュボードページ](https://dashboard.ngrok.com/get-started/setup)に移動し「Download for Mac OS」を押しダウンロードする
   ![](https://storage.googleapis.com/zenn-user-upload/1dbc1b894917-20220413.png)
3. その後は、[ダッシュボードページ](https://dashboard.ngrok.com/get-started/setup)記載の初期設定手順に従い設定を行う

## Ngrok の実装

まずターミナルを 2 つ準備する必要があります。
VSCode であれば以下のようにターミナルを画面分割で表示するとわかりやすいと思います。

![](https://storage.googleapis.com/zenn-user-upload/b0c13082502d-20220413.png)

1. Express を起動する
2. Ngrok を起動する

Ngrok を起動する際の注意点はポート番号の指定です。
Express のデフォルトポート番号を 5001 番に設定しているため、以下のようにコマンドを叩けば問題なく動きます。

```bash
ngrok http 5001
```

起動すると公開用の URL が発行されるのでこちらのドメイン + 相対パスを webhook に設定すれば、LINE Bot が機能します。

![](https://storage.googleapis.com/zenn-user-upload/4bb22921d07b-20220413.png)

`https://8e28278045ce.ngrok.io/line-bot`

## Ngrok のおすすめオプション（Ngrok の有料プランのみ）

### --subdomain

Ngrok はサブドメインを指定せずに実行すれば毎回ランダムの文字列がサブドメインに設定されます。
このオプションを使うことで、サブドメインをコピペする手間が省けます。
（私はそれだけのために課金する価値はあると考えてます）

### --region

Ngrok が起動するリージョンの指定ができます。

| コマンド | 場所名        |
| -------- | ------------- |
| `us`     | United States |
| `eu`     | Europe        |
| `ap`     | Asia/Pacific  |
| `au`     | Australia     |
| `sa`     | South America |
| `jp`     | Japan         |
| `in`     | India         |

デフォルトが`us`なため、ネットワーク回線の遅いところだと少し遅く感じるかもしれません。

## Ngrok と LINE Bot の連携

Ngrok と Express を起動し、URL をコピーしてください。

上記の設定を例にすると以下の URL になります。

```bash
https://8e28278045ce.ngrok.io/line-bot
```

先程作成した LINE Developers Console の Messaging API を開き、「Messaging API 設定」ページに移動します。

![](https://storage.googleapis.com/zenn-user-upload/edf0ba3c4ce1-20230909.png)

少し下に移動すると、「Webhook 設定」の項目があるので、こちらの URL に先程コピーした URL を貼り付け、「Webhook の利用」をオンにします。

![](https://storage.googleapis.com/zenn-user-upload/25fe71fc8795-20230909.png)

「Webhook URL を検証」を押すと、LINE Developers Console から Webhook に対してリクエストが送信されます。

正常に動作していれば以下のように表示されます。

![](https://storage.googleapis.com/zenn-user-upload/961ffd89f1a2-20230909.png)

以上で LINE Bot と Ngrok の連携は完了です。
