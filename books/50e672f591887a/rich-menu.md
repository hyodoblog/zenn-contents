---
title: "リッチメニューの設定"
---

こちらのセクションでは、リッチメニューの設定を行います。

## リッチメニューとは

リッチメニューとは、LINE のチャット画面上に表示されるメニューのことで、トーク画面下部（キーボードエリア）に固定で表示されるメニュー機能です。
リッチメニューには、クーポンやショップカードなどの LINE 公式アカウントの機能のほか、EC サイトや予約サイトなど、外部サイトへのリンクを設定できます。

参照
https://www.linebiz.com/jp/column/technique/20180731-01/

## リッチメニューを設定

本アプリでは以下の画像をリッチメニュー画像として利用します。（ご自身で用意しても大丈夫です）

こちらが用意したテンプレートをダウンロードしている場合、`assets/rich-menu.png`に保存されています。

![](https://storage.googleapis.com/zenn-user-upload/f35ba5335b3e-20230908.png)

以下のコマンドをターミナルで実行します。

```bash
yarn init:richmenu
```

作成した LINE 公式アカウントに移動し、リッチメニューが設定されていることを確認します。

## 解説

リッチメニューを設定するためのコードは`src/migrations/rich-menu/index.ts`にあり、以下がメイン関数です。

```ts
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

1. 登録中のリッチメニューを全て削除
2. 画像データのパスを設定
3. リッチメニューを作成

リッチメニューを作成する関数は以下のようになっています。

```ts
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
```

1. 画像データを Buffer で読み込み
2. リッチメニューの座標とアクションデータを設定
3. リッチメニューに画像をアタッチ

このような流れになってます。

リッチメニューは複数設定することもあるため、友だち登録時に表示するデフォルトのリッチメニューを設定するために`isDefault`という引数を設けて、`setDefaultRichMenu`関数でデフォルトのリッチメニューを設定しています。

### リッチメニューの座標とアクションデータ

`src/migrations/rich-menu/default.ts`ファイルにリッチメニューの座標とアクションデータが存在します。

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

リッチメニューは LINE Official Account Manager からも設定できますが、こちらの方法を使うとコードで管理できるため、開発効率が上がります。

また、`action`内の`postback`という機能は LINE Official Account Manager では設定できない機能なため、ソースコードで設定することが多いです。

ポストバックアクションについては以下を参照してください。

https://developers.line.biz/ja/docs/messaging-api/actions/#postback-action
