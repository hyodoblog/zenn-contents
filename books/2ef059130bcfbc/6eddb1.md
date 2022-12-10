---
title: "LINE Botの環境構築"
free: false
---

# 開発環境の構築

自前で容易するのが手間な方はこちらのテンプレートをお使いください。
オウム返しボットが構築されます。

https://zenn.dev/hyodoblog/articles/f3f97b25dc29c2

クローンが完了しましたら、以下のコマンドでnode_modulesをインストールしてください。
```bash
cd functions
yarn or npm i
```


# デバッグ環境の構築

デバッグにはNgrokを用います。
LINE Bot開発においてNgrokは大変強力なツールなので、それだけのために月額課金するものありだと私は考えてます。（課金ユーザーです）

https://zenn.dev/hyodoblog/articles/c1b2d6f6135af7


# Firebase Projectの新規作成

こちらのセクションは各々で準備をお願いします。

Firebase Projectが作成できたら、Firebase Project Idを`.firebaserc`ファイルに記載してください。

```json
{
  "projects": {
    "default": "ここにいれる"
  }
}
```

その他以下の設定をこの時点でお願いします。

- ロケーション設定
- Blazeプランへのアップグレード（FunctionsはBlazeプランでないと使用できないため）
- 予算設定（設定してないと知らない間にユーザーが増えてた際の支払いが多くなるため）


# LINE Messaging APIの初期設定

1. LINE Messaging APIをLINE Developer Consoleにて作成する。（この手順は省略します）
2. `チャネルアクセストークン`と`チャネルシークレット`をコピーする。 
3. `functions/.env.example`ファイルをコピーし、`functions/.env`ファイルを作成する。
4. `LINE_MESSAGING_CHANNEL_ACCESS_TOKEN`に`チャネルアクセストークン`を、 `LINE_MESSAGING_CHANNEL_SECRET`に`チャネルシークレット`を設定する。


# デプロイ

デプロイ方法は以下のコマンドで実行できます。
⚠`functions`パス内で実行すること

```bash
yarn build && firebase deploy
or 
npm run build && firebase deploy
```

この

--- ここまでのコード ---
https://github.com/hyodoblog/zenn-mojiokoshi-varygoodkun/commit/e214022f7b34989d6088fe260e82728cfc8fa3d3