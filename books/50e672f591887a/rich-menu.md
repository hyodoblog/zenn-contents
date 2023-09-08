---
title: "リッチメニューの設定"
---

## リッチメニューとは

リッチメニューとは、LINE のチャット画面上に表示されるメニューのことで、トーク画面下部（キーボードエリア）に固定で表示されるメニュー機能です。
リッチメニューには、クーポンやショップカードなどの LINE 公式アカウントの機能のほか、EC サイトや予約サイトなど、外部サイトへのリンクを設定できます。

参照
https://www.linebiz.com/jp/column/technique/20180731-01/

## リッチメニューを設定

本アプリでは以下の画像をリッチメニュー画像として利用します。（ご自身で用意しても大丈夫です）
保存してください。

![](https://storage.googleapis.com/zenn-user-upload/f35ba5335b3e-20230908.png)

`src/migrations/rich-menu/index.ts`ファイルを作成し、以下のコードを記述します。

```ts
import "../../alias";

import { RichMenu } from "@line/bot-sdk";
import { readFileSync } from "fs";
import { join } from "path";
import { lineClient } from "~/clients/line.client";
import { defaultRichMenu } from "./default";

const allDeleteRichmenu = async (): Promise<void> => {
  const richmenuIds: string[] = (await lineClient.getRichMenuList()).map(
    (value) => value.richMenuId
  );
  await Promise.all(
    richmenuIds.map((richmenuId) => lineClient.deleteRichMenu(richmenuId))
  );
};

const createRichmenu = async (
  imgPath: string,
  richmenu: RichMenu,
  isDefault = false
): Promise<string> => {
  const imgFile = readFileSync(imgPath);

  const richmenuId = await lineClient.createRichMenu(richmenu);
  await lineClient.setRichMenuImage(richmenuId, imgFile);

  if (isDefault) {
    await lineClient.setDefaultRichMenu(richmenuId);
  }

  return richmenuId;
};

(async () => {
  try {
    await allDeleteRichmenu();

    const imgPath = join(__dirname, "../../../assets/rich-menu.png");
    await createRichmenu(imgPath, defaultRichMenu, true);

    console.info("finish.");
  } catch (err) {
    console.error(err);
  }
})();
```

`src/migrations/rich-menu/default.ts`ファイルを作成し、以下のコードを記述します。

```ts
import { RichMenu } from "@line/bot-sdk";

export const defaultRichMenu: RichMenu = {
  size: {
    width: 1200,
    height: 300,
  },
  selected: true,
  name: "リッチメニュー",
  chatBarText: "注文&マイページはこちら👇",
  areas: [
    {
      bounds: {
        x: 0,
        y: 0,
        width: 300,
        height: 300,
      },
      action: {
        type: "postback",
        text: "商品一覧を見る",
        data: "products",
      },
    },
    {
      bounds: {
        x: 301,
        y: 0,
        width: 300,
        height: 300,
      },
      action: {
        type: "postback",
        text: "注文履歴&マイページを見る",
        data: "mypage",
      },
    },
    {
      bounds: {
        x: 601,
        y: 0,
        width: 300,
        height: 300,
      },
      action: {
        type: "uri",
        uri: "https://yoshinani.dev",
      },
    },
    {
      bounds: {
        x: 901,
        y: 0,
        width: 300,
        height: 300,
      },
      action: {
        type: "uri",
        uri: "https://twitter.com/hyodoblog?openExternalBrowser=1",
      },
    },
  ],
};
```

以下のコマンドをターミナルで実行します。

```bash
yarn init:richmenu
```

作成した LINE 公式アカウントに移動し、リッチメニューが設定されていることを確認します。
