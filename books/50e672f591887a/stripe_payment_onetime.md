---
title: "単発購入処理の実装"
---

こちらのセクションでは、単発購入処理の実装を紹介します。

## 単発購入関数の紹介

単発購入処理の実装は、Stripe Checkout Session API を使って実装します。

まず、Stripe Checkout Session API で決済 URL を発行する方法を紹介します。

```ts
const purchase = async (
  customerId: string,
  priceId: string
): Promise<{ url: string }> => {
  const { url } = await stripeClient.checkout.sessions.create({
    customer: customerId,
    line_items: [
      {
        price: priceId,
        quantity: 1,
      },
    ],
    shipping_address_collection: {
      allowed_countries: ["JP"],
    },
    invoice_creation: {
      enabled: true,
    },
    mode: "payment",
    payment_method_types: ["card"],
    success_url: LINE_FRIEND_URL,
    cancel_url: LINE_FRIEND_URL,
  });

  if (url === null) {
    throw new Error("url is null");
  }

  return { url };
};
```

この関数は、`stripe customer id` と `stripe price id` を受け取って、決済 URL を返します。

- `shipping_address_collection`オプションを指定することで、住所の入力を求めることができます。
- `invoice_create.enable`オプションを true にすることで、決済完了後に請求書を作成することができます。こちらは、Billing Portal の履歴に表示させるために必要です。
- `success_url`と`cancel_url`に友だち登録 URL を指定することで、決済完了後または中断してもトーク画面に戻ることができます。

## 実装

それでは、実際に実装していきます。

`src/routes/line-bot/handlers/postback/products/one-time.ts`

```ts
import { PostbackEvent } from "@line/bot-sdk";
import { lineClient } from "~/clients/line.client";
import { errorConsole } from "~/utils/util";
import { msgPurchase } from "~/notice-messages/purchase";
import { stripeClient } from "~/clients/stripe.client";
import { getCustomer } from "~/domains/customer.domain";
import { LINE_FRIEND_URL } from "~/utils/secrets";

const purchase = async (
  customerId: string,
  priceId: string
): Promise<{ url: string }> => {
  const { url } = await stripeClient.checkout.sessions.create({
    customer: customerId,
    line_items: [
      {
        price: priceId,
        quantity: 1,
      },
    ],
    shipping_address_collection: {
      allowed_countries: ["JP"],
    },
    invoice_creation: {
      enabled: true,
    },
    mode: "payment",
    payment_method_types: ["card"],
    success_url: LINE_FRIEND_URL,
    cancel_url: LINE_FRIEND_URL,
  });

  if (url === null) {
    throw new Error("url is null");
  }

  return { url };
};

export const postbackProductsOneTimeHandler = async (
  event: PostbackEvent,
  priceId: string
): Promise<void> => {
  try {
    const customer = await getCustomer(event.source.userId!);
    const { url } = await purchase(customer.id, priceId);

    await lineClient.replyMessage(event.replyToken, msgPurchase(url));
  } catch (err) {
    errorConsole(err);
    throw new Error("postback products one-time handler");
  }
};
```
