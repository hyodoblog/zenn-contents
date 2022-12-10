---
title: "【VSCode】code-workspaceファイルのエイリアスをデスクトップに置くと開発速度が上がった"
emoji: "😎"
type: "idea"
topics:
  - "vscode"
published: true
published_at: "2022-04-19 18:48"
---

**本記事は、開発エディターにVSCodeを採用している方を対象としております**


# VSCodeワークスペース

VSCodeには、ワークスペースという機能がありソースコードでVSCodeの設定ファイルを組織またはチームごとに統一することができます。

詳しくはこちらの記事を参考ください。
https://qiita.com/YuichiNukiyama/items/ef16a0219f46ea03a045#%E3%83%AF%E3%83%BC%E3%82%AF%E3%82%B9%E3%83%9A%E3%83%BC%E3%82%B9%E3%81%A8%E3%81%AF


# `.code-workspace`ファイルをデスクトップに置く利点

- パソコンを起動した際、デスクトップ画面には開発環境すばやくアクセスするためのものが並んでいるため、すぐに仕事に取り掛かることができるため、無駄な時間を省くことができる
- monorepo開発時、`folders`でプロジェクトを区切ることでファイル検索&アクセスが容易になる
- プロジェクトのチームごとにVSCodeの設定が共有できる
- 過去のコードを参考にしたいとき、すぐにファイルを開くことができる


# 手順

1. `.code-workspace`ファイルを作成し、プロジェクトのトップに置く
2. `.code-workspace`ファイルのショートカットファイルをデスクトップに置く

https://applica.info/mac-shortcut-create


## `.code-workspace`の書き方

基本の書き方

```json
{
  "folders": [
    {
      "path": "."
    },
  ],
  "settings": {}
}
```

`folders`には、プロジェクトのトップの相対パスを設定します。
monorepo開発している際`folders`の設定はかなりの効果を発揮しますが、単一プロジェクトの場合は一つのことが多いです。

`settings`には、VSCodeの環境設定を設定できます。
例えばeslintを保存と同時に実行したい場合は、以下のように記載すれば他のチームメンバーにも同じ設定ファイルが共有されます。

```json
{
  "folders": [
    {
      "path": "."
    },
  ],
  "settings": {
    "editor.codeActionsOnSave": {
      "source.fixAll.eslint": true
    }
  }
}
```


## monorepo開発

```json
{
  "folders": [
    {
      "path": "."
    },
    {
      "name": "client",
      "path": "./pacakges/nextjs"
    },
    {
      "name": "server",
      "path": "./packages/nestjs"
    },
  ],
  "settings": {
    "editor.codeActionsOnSave": {
      "source.fixAll.eslint": true
    }
  }
}
```

monorepo開発時は、`packages`ディレクトリで区切ることが多いと思います。
今回の例は、`client`と`server`の2つのプロジェクトをmonorepoで開発しているときの設定方法です。

このように設定することで、VSCodeのファイル検索機能（cmd + p）で`package.json`ファイルにアクセスしたいとき、`package client`と入力し`Enter`キーを押せば、`client`の`package.json`に容易にアクセスできます。`server`の場合、`package server`と入力します。

**ファイルへのアクセス速度が上がれば必然的に開発速度もあがります！**


# 参考

こちらが執筆時の著者のデスクトップ画面です。

![](https://storage.googleapis.com/zenn-user-upload/b95a8e10b632-20220419.png =500x)

執筆時の著者は、LINE関係のアプリまたはサービスしか開発していないため、直近で開発したLINE関係のVSCodeファイルのみをデスクトップにおいています。

*⚠作ったものすべてを置いてしまうと、探すのが大変で、過去の汚いコードにアクセスすることがあるため、厳選してデスクトップに置きましょう。（10~20ファイルまでがちょうどいいと思います）*


# まとめ

このように、`.code-workspace`ファイルをデスクトップに置くことで、直近のプロジェクトと過去のコードへのアクセス速度が上がるため、開発速度を向上させることができます。
また、ワークスペースはチーム開発でも強力なVSCodeの機能で、組織やプロジェクトに応じて開発環境を統一できます。

ぜひ、使ってみてください❗