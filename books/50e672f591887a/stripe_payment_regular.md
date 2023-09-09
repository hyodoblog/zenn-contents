---
title: "定期購入処理の実装"
---

こちらのセクションでは、定期購入処理の実装を紹介します。

## 定期購入関数の紹介

定期購入処理の実装は、Stripe Checkout Session API を使って実装します。

まず、Stripe Checkout Session API て決済 URL を発行する方法を紹介します。

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
    mode: "subscription",
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
- `success_url`と`cancel_url`に友だち登録 URL を指定することで、決済完了後または中断してもトーク画面に戻ることができます。

## 実装

それでは、実際に実装していきます。

`src/routes/line-bot/handlers/postback/products/regular.ts`

```ts
import { PostbackEvent } from "@line/bot-sdk";
import { lineClient } from "~/clients/line.client";
import { errorConsole } from "~/utils/util";
import { msgPurchase } from "~/notice-messages/purchase";
import { LINE_FRIEND_URL } from "~/utils/secrets";
import { stripeClient } from "~/clients/stripe.client";
import { getCustomer } from "~/domains/customer.domain";

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
    mode: "subscription",
    payment_method_types: ["card"],
    success_url: LINE_FRIEND_URL,
    cancel_url: LINE_FRIEND_URL,
  });

  if (url === null) {
    throw new Error("url is null");
  }

  return { url };
};

export const postbackProductsRegularHandler = async (
  event: PostbackEvent,
  priceId: string
): Promise<void> => {
  try {
    const customer = await getCustomer(event.source.userId!);
    const { url } = await purchase(customer.id, priceId);

    await lineClient.replyMessage(event.replyToken, msgPurchase(url));
  } catch (err) {
    errorConsole(err);
    throw new Error("postback products regular handler");
  }
};
```

`src/routes/line-bot/handlers/postback/products/index.ts`ファイルを以下のように編集します。

```ts
import { PostbackEvent } from "@line/bot-sdk";
import { errorConsole } from "~/utils/util";
import { postbackProductsDetailHandler } from "./detail";
import { postbackProductsOneTimeHandler } from "./one-time";
import { postbackProductsListHandler } from "./list";
import { postbackProductsRegularHandler } from "./regular"; /* -- 追加 -- */

export const postbackProductsHandler = async (
  event: PostbackEvent
): Promise<void> => {
  try {
    const { data } = event.postback;

    if (data === "products") {
      return await postbackProductsListHandler(event);
    }

    if (data.includes("products.")) {
      const [, productType, priceId] = data.split(".");
      switch (productType) {
        case "detail":
          return await postbackProductsDetailHandler(event, priceId);
        case "one-time":
          return await postbackProductsOneTimeHandler(event, priceId);
        /* -- 追加 -- */
        case "regular":
          return await postbackProductsRegularHandler(event, priceId);
        /* --------- */
      }
    }
  } catch (err) {
    errorConsole(err);
    throw new Error("postback products handler");
  }
};
```

これで商品一覧の「詳細を見る」ボタンを押した後、「定期購入する」ボタンを押すと、定期購入の決済画面に遷移します。

![](https://storage.googleapis.com/zenn-user-upload/9bbb6b3397fe-20230909.jpg =300x)

以上で、定期購入処理の実装は完了です。
