---
title: "商品一覧の実装"
---

## 商品を追加

まず初めに Stripe の「商品」に商品情報を追加します。

1. [こちら](https://dashboard.stripe.com/test/products?active=true)にアクセスし、「商品を追加」をクリックします。
   ![](https://storage.googleapis.com/zenn-user-upload/49d6982f4676-20230908.png)

2. 基本情報を設定
   - 名前と画像を設定
3. 単発料金を設定
   - 価格を設定
   - 「一括」
4. 定期料金を設定
   - 「別の料金を追加」をクリック
   - 価格を設定
   - 「継続」を選択
5. 「商品を保存する」をクリック

そうすると商品詳細ページの「料金」箇所に、先ほど設定した料金がそれぞれ表示されます。

![](https://storage.googleapis.com/zenn-user-upload/f4f8b2afc3ce-20230908.png)

これで商品追加は完了です。

::: message
追加で商品を作成していただいて大丈夫です。
現状は 10 個以内の商品を作成していただければ大丈夫です。
本アプリで商品一覧に使用しているカールセルは、10 個までしか表示できないためです。
:::

## 実装

### 商品一覧取得の実装

Stripe に登録した商品情報を取得するには、`Stripe Product API`を使用します。
`src/domains/product.domain.ts`ファイルを作成し以下のコードを記述します。

```ts
import Stripe from "stripe";
import { stripeClient } from "~/clients/stripe.client";

export const getProducts = async (): Promise<Stripe.Product[]> => {
  const { data } = await stripeClient.products.list();
  return data;
};
```

`getProducts`関数を使えば、Stripe ダッシュボードに追加した商品情報を一括で取得ができます。

次は、LINE 公式アカウントから商品情報を取得できるようにします。
商品情報を取得する関数のために以下のファイルを作成してください。

- `src/routes/line-bot/handlers/postback/index.ts`
- `src/routes/line-bot/handlers/postback/products/index.ts`
- `src/routes/line-bot/handlers/postback/products/list.ts`
- `src/notice-messages/products.ts`

それぞれのファイルの役割は以下の通りです。

- `postback/index.ts`：`postback`イベントを処理する
- `postback/products/index.ts`：商品関係の関数を呼び出す
- `postback/products/list.ts`：商品一覧を取得する
- `notice-messages/products.ts`：商品一覧を表示する LINE フレックスメッセージ

それではコードを記述していきます。

`src/notice-messages/products.ts`ファイルに以下のコードを記述します。

```ts
import { FlexBubble, FlexCarousel, FlexMessage } from "@line/bot-sdk";

export interface MsgProductList {
  productId: string;
  priceId: string;
  name: string;
  imgUrl: string;
  amount: number;
}

export const msgProducts = (products: MsgProductList[]): FlexMessage => {
  const productContents: FlexBubble[] = [];

  for (const product of products) {
    productContents.push({
      type: "bubble",
      size: "kilo",
      hero: {
        type: "image",
        url: product.imgUrl,
        size: "full",
        aspectRatio: "1:1",
        aspectMode: "cover",
      },
      body: {
        type: "box",
        layout: "vertical",
        spacing: "sm",
        contents: [
          {
            type: "text",
            text: product.name,
            weight: "bold",
            size: "xl",
            wrap: false,
          },
          {
            type: "box",
            layout: "baseline",
            contents: [
              {
                type: "text",
                text: `¥${Number(product.amount).toLocaleString()}`,
                weight: "bold",
                size: "xl",
                flex: 0,
                wrap: true,
              },
              {
                type: "text",
                text: "(税込)",
                weight: "bold",
                size: "sm",
                flex: 0,
                margin: "md",
                wrap: true,
              },
            ],
          },
        ],
      },
      footer: {
        type: "box",
        layout: "vertical",
        spacing: "md",
        contents: [
          {
            type: "button",
            action: {
              type: "postback",
              label: "単体で購入する",
              displayText: "単体で購入する。",
              data: `products.one-time.${product.priceId}`,
            },
            color: "#003CF0FF",
            style: "primary",
          },
          {
            type: "button",
            action: {
              type: "postback",
              label: "詳細を見る",
              displayText: "詳細を見る。",
              data: `products.detail.${product.productId}`,
            },
            style: "secondary",
          },
        ],
      },
    });
  }

  const contents: FlexCarousel = {
    type: "carousel",
    contents: productContents,
  };

  return {
    type: "flex",
    altText: "商品一覧を見る",
    contents,
  };
};
```

`src/routes/line-bot/handlers/postback/products/list.ts`ファイルに以下のコードを記述します。

```ts
import { PostbackEvent } from "@line/bot-sdk";
import { lineClient } from "~/clients/line.client";
import { errorConsole } from "~/utils/util";
import { MsgProductList, msgProducts } from "~/notice-messages/products";
import { getProducts } from "~/domains/product.domain";
import { stripeClient } from "~/clients/stripe.client";

export const postbackProductsListHandler = async (
  event: PostbackEvent
): Promise<void> => {
  try {
    const products = await getProducts();

    const _products: MsgProductList[] = [];

    await Promise.all(
      products.map(async (product) => {
        if (typeof product.default_price !== "string") {
          return;
        }

        const price = await stripeClient.prices.retrieve(product.default_price);
        _products.push({
          productId: product.id,
          priceId: price.id,
          name: product.name,
          imgUrl: product.images[0],
          amount: Number(price.unit_amount),
        });
      })
    );

    await lineClient.replyMessage(event.replyToken, msgProducts(_products));
  } catch (err) {
    errorConsole(err);
    throw new Error("postback products list handler");
  }
};
```

`src/routes/line-bot/handlers/postback/products/index.ts`ファイルに以下のコードを記述します。

```ts
import { PostbackEvent } from "@line/bot-sdk";
import { errorConsole } from "~/utils/util";
import { postbackProductsListHandler } from "./list";

export const postbackProductsHandler = async (
  event: PostbackEvent
): Promise<void> => {
  try {
    const { data } = event.postback;

    if (data === "products") {
      return await postbackProductsListHandler(event);
    }
  } catch (err) {
    errorConsole(err);
    throw new Error("postback products handler");
  }
};
```

`src/routes/line-bot/handlers/postback/index.ts`ファイルに以下のコードを記述します。

```ts
import { PostbackEvent } from "@line/bot-sdk";
import { errorConsole } from "~/utils/util";
import { postbackMypageHandler } from "./mypage";
import { postbackProductsHandler } from "./products";

export const postbackHandler = async (event: PostbackEvent): Promise<void> => {
  try {
    const { data } = event.postback;
    if (data.includes("products")) {
      return await postbackProductsHandler(event);
    }
  } catch (err) {
    errorConsole(err);
    throw new Error("postback handler");
  }
};
```

次に、`postback`処理を呼び出すために`src/routes/line-bot/handlers/index.ts`ファイルを以下のように編集します。
`postbackHandler`関数を追加します。

```ts
import { WebhookEvent } from "@line/bot-sdk";
import { lineClient } from "~/clients/line.client";
import { msgError } from "~/notice-messages/error";

import { followHandler } from "./follow";
import { errorConsole } from "~/utils/util";
import { postbackHandler } from "./postback";

export const handlers = async (event: WebhookEvent): Promise<void> => {
  try {
    switch (event.type) {
      case "follow":
        return await followHandler(event);
      case "postback":
        return await postbackHandler(event);
    }
  } catch (err) {
    lineClient.pushMessage(event.source.userId!, msgError).catch;
    errorConsole(err);
    throw new Error("handlers");
  }
};
```

これでリッチメニューの「商品一覧」ボタンを押したとき、Stripe の商品に登録した情報が以下のように商品一覧が表示されます。

![](https://storage.googleapis.com/zenn-user-upload/e6b48ae1fb18-20230909.jpg =300x)

### 商品詳細取得の実装

次に商品詳細を表示するために、以下のファイルを作成してください。

- `src/routes/line-bot/handlers/postback/products/detail.ts`
- `src/notice-messages/product.ts`

それぞれのファイルの役割は以下の通りです。

- `postback/products/detail.ts`：商品詳細を取得する
- `notice-messages/product.ts`：商品詳細を表示する LINE フレックスメッセージ

それではコードを記述していきます。

`src/notice-messages/product.ts`ファイルに以下のコードを記述します。

```ts
import { FlexBubble, FlexComponent, FlexMessage } from "@line/bot-sdk";

export interface MsgProduct {
  name: string;
  imgUrl: string;
  description: string;
  goodAmount: number | null;
  serviceAmount: number | null;
  goodPriceId: string | null;
  servicePriceId: string | null;
}

export const msgProduct = (product: MsgProduct): FlexMessage => {
  const footerContents: FlexComponent[] = [];
  if (product.goodPriceId !== null && product.goodAmount !== null) {
    footerContents.push({
      type: "button",
      action: {
        type: "postback",
        label: `単体で購入する(¥${Number(
          product.goodAmount
        ).toLocaleString()})`,
        text: "単体で購入する。",
        data: `products.one-time.${product.goodPriceId}`,
      },
      color: "#003CF0",
      style: "primary",
    });
  }
  if (product.servicePriceId !== null && product.serviceAmount !== null) {
    footerContents.push({
      type: "button",
      action: {
        type: "postback",
        label: `定期購入する(¥${Number(
          product.serviceAmount
        ).toLocaleString()})`,
        text: "定期購入する。",
        data: `products.regular.${product.servicePriceId}`,
      },
      color: "#001E77",
      style: "primary",
    });
  }

  const contents: FlexBubble = {
    type: "bubble",
    size: "giga",
    direction: "ltr",
    hero: {
      type: "image",
      url: product.imgUrl,
      size: "full",
      aspectRatio: "1.51:1",
      aspectMode: "fit",
    },
    body: {
      type: "box",
      layout: "vertical",
      contents: [
        {
          type: "spacer",
          size: "xs",
        },
        {
          type: "text",
          text: product.name,
          weight: "bold",
          size: "xl",
          align: "start",
        },
        {
          type: "text",
          text: product.description,
          align: "start",
          wrap: true,
        },
        {
          type: "spacer",
          size: "xs",
        },
      ],
    },
    footer: {
      type: "box",
      layout: "vertical",
      spacing: "md",
      margin: "md",
      contents: footerContents,
    },
  };

  return {
    type: "flex",
    altText: "商品詳細を見る",
    contents,
  };
};
```

`src/routes/line-bot/handlers/postback/products/detail.ts`ファイルに以下のコードを記述します。

```ts
import { PostbackEvent } from "@line/bot-sdk";
import Stripe from "stripe";
import { lineClient } from "~/clients/line.client";
import { errorConsole } from "~/utils/util";
import { msgProduct, MsgProduct } from "~/notice-messages/product";
import { stripeClient } from "~/clients/stripe.client";
import { getPricesByProductId } from "~/domains/price.domain";

const getGoogItemByPrices = (
  prices: Stripe.Price[]
): { goodPriceId: string | null; goodPriceAmount: number | null } => {
  const price = prices.filter((price) => price.type === "one_time")[0];
  if (price === undefined) {
    return { goodPriceId: null, goodPriceAmount: null };
  } else {
    return { goodPriceId: price.id, goodPriceAmount: price.unit_amount };
  }
};

const getServiceItemByPrices = (
  prices: Stripe.Price[]
): { servicePriceId: string | null; servicePriceAmount: number | null } => {
  const price = prices.filter((price) => price.type === "recurring")[0];
  if (price === undefined) {
    return { servicePriceId: null, servicePriceAmount: null };
  } else {
    return { servicePriceId: price.id, servicePriceAmount: price.unit_amount };
  }
};

export const postbackProductsDetailHandler = async (
  event: PostbackEvent,
  productId: string
): Promise<void> => {
  try {
    const _product = await stripeClient.products.retrieve(productId);
    const prices = await getPricesByProductId(productId);
    const goodItem = getGoogItemByPrices(prices);
    const serviceItem = getServiceItemByPrices(prices);

    const product: MsgProduct = {
      name: _product.name,
      imgUrl: _product.images[0],
      description: _product.description || "説明がありません。",
      goodAmount: goodItem.goodPriceAmount,
      serviceAmount: serviceItem.servicePriceAmount,
      goodPriceId: goodItem.goodPriceId,
      servicePriceId: serviceItem.servicePriceId,
    };

    await lineClient.replyMessage(event.replyToken, msgProduct(product));
  } catch (err) {
    errorConsole(err);
    throw new Error("postback products detail handler");
  }
};
```

`src/routes/line-bot/handlers/postback/products/index.ts`ファイルに以下のコードを上書きします。

```ts
import { PostbackEvent } from "@line/bot-sdk";
import { errorConsole } from "~/utils/util";
import { postbackProductsDetailHandler } from "./detail";
import { postbackProductsListHandler } from "./list";

export const postbackProductsHandler = async (
  event: PostbackEvent
): Promise<void> => {
  try {
    const { data } = event.postback;

    if (data === "products") {
      return await postbackProductsListHandler(event);
    } else if (data.includes("products.")) {
      const [, productType, priceId] = data.split(".");
      switch (productType) {
        case "detail":
          return await postbackProductsDetailHandler(event, priceId);
      }
    }
  } catch (err) {
    errorConsole(err);
    throw new Error("postback products handler");
  }
};
```

これで商品一覧の「詳細を見る」ボタンを押すと、Stripe に登録した商品の詳細が表示されます。

![](https://storage.googleapis.com/zenn-user-upload/044f92ddef3a-20230909.jpg =300x)

それでは次から決済処理を実装していきます。
