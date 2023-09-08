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

### LINE 公式アカウントと連携

次は、LINE 公式アカウントから商品情報を取得できるようにします。

商品情報を取得する関数のために以下のファイルを作成してください。

- `src/routes/line-bot/handlers/postback/index.ts`
- `src/routes/line-bot/handlers/postback/products/index.ts`
- `src/routes/line-bot/handlers/postback/products/list.ts`

それぞれのファイルの役割は以下の通りです。

- `postback/index.ts`：`postback`イベントを処理する
- `postback/products/index.ts`：商品関係の関数を呼び出す
- `postback/products/list.ts`：商品一覧を取得する

それではコードを記述していきます。

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
