---
title: "Stripe の初期設定"
---

このセクションでは、Stripe の初期設定を行います。

## Stripe の初期設定

Stripe API を利用するための環境を構築します。

`src/clients/stripe.client.ts`ファイルを作成し以下のコードを記述します。

```ts
import Stripe from "stripe";
import { STRIPE_SECRET_KEY } from "~/utils/secrets";

export const stripeClient = new Stripe(STRIPE_SECRET_KEY, {
  apiVersion: "2023-08-16",
});
```

stripe の`apiVersion`は年に 1,2 回更新があります。

ライブラリーを最新版にしても、こちらのバージョンを変更しない限り最新版の API を利用することはできないため新機能の利用には注意が必要です。

以降のセクションでは、`stripeClient`メソッドを使って Stripe API を利用します。
