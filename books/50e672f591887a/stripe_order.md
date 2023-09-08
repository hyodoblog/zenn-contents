---
title: "注文履歴の実装"
---

このセクションでは、注文履歴を実装します。

## Billing カスタマーポータル

本アプリでは、Stripe の Billing カスタマーポータル を利用して、顧客が自分の情報を確認したり、支払い方法を変更したりできるようにします。

Billing カスタマーポータルは Stripe が提供する製品であり、顧客が各自で支払い方法の更新、請求書の管理、サブスクリプションの管理、支払い履歴の表示などをできるようにするものです。この機能は有料の Billing のユーザーに提供されます。

参照
https://support.stripe.com/questions/billing-customer-portal?locale=ja-JP

Billing カスタマーポータルを利用するには、Stripe の API を利用して、顧客の ID を指定して、Billing カスタマーポータルの URL を取得する必要があります。

参照
https://stripe.com/docs/customer-management

## Billing カスタマーポータルの実装

```ts
const { url } = await stripeClient.billingPortal.sessions.create({
  customer: customer.id,
  return_url: LINE_FRIEND_URL,
});
```

Billing カスタマーポータルの URL を取得するには、上記のように `billingPortal.sessions.create` を利用します。

こちらを LINE に組み込んでいきます。

`src/routes/line-bot/handlers/postback/mypage.ts`ファイルを作成し、以下のように実装します。

```ts
import { PostbackEvent } from "@line/bot-sdk";
import { lineClient } from "~/clients/line.client";
import { LINE_FRIEND_URL } from "~/utils/secrets";
import { errorConsole } from "~/utils/util";
import { msgMypage } from "~/notice-messages/mypage";
import { getCustomer } from "~/domains/customer.domain";
import { stripeClient } from "~/clients/stripe.client";

export const postbackMypageHandler = async (
  event: PostbackEvent
): Promise<void> => {
  try {
    const customer = await getCustomer(event.source.userId!);
    const { url } = await stripeClient.billingPortal.sessions.create({
      customer: customer.id,
      return_url: LINE_FRIEND_URL,
    });

    await lineClient.replyMessage(event.replyToken, msgMypage(url));
  } catch (err) {
    errorConsole(err);
    throw new Error("postback products handler");
  }
};
```

`src/notice-messages`ファイルに以下を追記します。

```ts

```
