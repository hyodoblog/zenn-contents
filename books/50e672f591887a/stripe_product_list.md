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

### LINE 上から取得できるようにする
