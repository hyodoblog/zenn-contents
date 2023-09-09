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

`src/domains/customer.domain.ts`ファイルを作成し、以下のように実装します。

```ts
import Stripe from "stripe";
import { lineClient } from "~/clients/line.client";
import { stripeClient } from "~/clients/stripe.client";

export const getCustomer = async (userId: string): Promise<Stripe.Customer> => {
  const { data } = await stripeClient.customers.search({
    query: `metadata['userId']:'${userId}'`,
  });
  if (data.length === 0) {
    const lineProfile = await lineClient.getProfile(userId);
    return await stripeClient.customers.create({
      name: lineProfile.displayName || "未設定",
      description: userId,
      metadata: {
        userId,
      },
    });
  } else {
    return data[0];
  }
};
```

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

`src/routes/line-bot/handlers/postback/products/index.ts`ファイルを以下のように編集する。

```ts
import { PostbackEvent } from "@line/bot-sdk";
import { errorConsole } from "~/utils/util";
import { postbackProductsDetailHandler } from "./detail";
import { postbackProductsOneTimeHandler } from "./one-time"; /* -- 追加 -- */
import { postbackProductsListHandler } from "./list";

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
        /* -- 追加 -- */
        case "one-time":
          return await postbackProductsOneTimeHandler(event, priceId);
        /* --------- */
      }
    }
  } catch (err) {
    errorConsole(err);
    throw new Error("postback products handler");
  }
};
```

`src/notice-messages/purchase.ts`ファイルを作成し、以下のように実装します。

```ts
import { FlexBubble, FlexMessage } from "@line/bot-sdk";

export const msgPurchase = (uri: string): FlexMessage => {
  const contents: FlexBubble = {
    type: "bubble",
    direction: "ltr",
    header: {
      type: "box",
      layout: "vertical",
      contents: [
        {
          type: "text",
          text: "有効期限は1時間です。",
          weight: "regular",
          align: "center",
          wrap: true,
        },
      ],
    },
    footer: {
      type: "box",
      layout: "horizontal",
      contents: [
        {
          type: "button",
          action: {
            type: "uri",
            label: "決済ページへ",
            uri,
          },
          color: "#FF6B00",
          style: "primary",
        },
      ],
    },
    styles: {
      header: {
        separator: false,
      },
      footer: {
        separator: false,
      },
    },
  };

  return {
    type: "flex",
    altText: "購入リンク",
    contents,
  };
};
```

これで、単発購入処理の実装は完了です。
商品一覧の「単発で購入する」ボタンより、決済画面に遷移することができます。

![](https://storage.googleapis.com/zenn-user-upload/734fcae90aa9-20230909.jpg =300x)

以上で、単発決済の実装は完了です。

ここまでのコードは、以下のコミットで確認できます。

https://github.com/hyodoblog/line-stripe-not-db-ec/commit/7a7eee822a73ef3c1b7d6990beada6bda86417dc
