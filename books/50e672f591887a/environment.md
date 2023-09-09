---
title: "プロジェクトの初期化と環境変数の用意"
---

## プロジェクトの初期化

まずは、プロジェクトを初期化します。

以下のリポジトリよりソースコードをダウンロードしてください。

https://github.com/hyodoblog/line-stripe-not-db-ec/releases/tag/init

## 環境変数を設定

`.env.example`ファイルをコピーし`.env`ファイルを作成する。

- LINE_MESSAGING_CHANNEL_ACCESS_TOKEN
- LINE_MESSAGING_CHANNEL_SECRET
- LINE_FRIEND_URL
- STRIPE_SECRET_KEY

上記 4 つのキーを LINE Developers Console と Stripe Dashboard から取得します。

## LINE Messaging API トークン発行方法

https://www.youtube.com/watch?v=lQivbzuYdiM

1. [LINE Developer Console](https://developers.line.biz/console/)にアクセス
2. プロバイダーを作成
3. Messaging API を作成
4. チャネルアクセストークンとチャネルシークレットを発行

## LINE 公式アカウントの友だち追加 URL を取得

1. ![](https://storage.googleapis.com/zenn-user-upload/22fc74438a31-20230908.png)

2. ![](https://storage.googleapis.com/zenn-user-upload/4f71d255a4d5-20230908.png)

3. ![](https://storage.googleapis.com/zenn-user-upload/b9d5a25adb84-20230908.png)

4. `.env`ファイル内の`LINE_FRIEND_URL`に設定

## Stripe シークレットキーの発行方法

https://www.youtube.com/watch?v=AmguoMbHQy4

1. [Stripe Dashboard](https://dashboard.stripe.com/dashboard/)にアクセス
2. アカウントを作成
3. シークレットキーを発行
4. アクセストークンを発行
5. `.env`ファイル内の`LINE_MESSAGING_CHANNEL_SECRET`と`LINE_MESSAGING_CHANNEL_ACCESS_TOKEN`に設定

上記で発行したキーを以下に設定します。

```bash
LINE_MESSAGING_CHANNEL_ACCESS_TOKEN=
LINE_MESSAGING_CHANNEL_SECRET=
LINE_FRIEND_URL=

STRIPE_SECRET_KEY=
```

以上で環境構築は完了です。

## ローカルでの動作確認

環境変数が設定完了したら、ローカルで動作確認を行います。
以下のコマンドをターミナルに入力してください。

```bash
npm run dev
```

以下のようなログが出力されたら成功です。

```bash
App listening at http://localhost:5001
```

それでは中身の実装を進めていきます。
