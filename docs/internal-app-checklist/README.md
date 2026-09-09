# 社内アプリ公開・認証チェックリスト

社内向けのWebアプリをVercel等へ公開するときに、公開範囲・認証・認可・秘密情報の設定漏れを確認するためのチェックリスト。

## 収録物

| ファイル | 中身 |
| --- | --- |
| [`checklist.md`](./checklist.md) | 9分野・66項目の詳細版。確認項目と確認方法をセットで記載 |

## 関連記事

- [Claude CodeやCodexで社内アプリを作るときの注意点](https://zenn.dev/yoshinani_dev/articles/3306277f57fd1c) — noindexをアクセス制御と取り違えた記録。Deployment Protectionの適用範囲、Vercelの生成URL、proxyでのBasic認証
- [Better Authの組織プラグインで社内アプリの認証を作った](https://zenn.dev/yoshinani_dev/articles/762690f92d33f8) — 招待制とロール、認可の置き場所、ライブラリの標準APIを絞る話
- [アプリが3つになったので自前のIdPを立てた](https://zenn.dev/yoshinani_dev/articles/d131524e17279e) — 複数アプリで認証を集約する。認証経路を1本に絞り、会員判定をセッション生成前へ置く

## 位置付け

より広い範囲を扱う[システムセキュリティチェックリスト](../security-checklist/checklist.md)（15分野・140項目）を、社内アプリの公開範囲と認証に絞って具体化したもの。両方を使う場合は、先にこちらで公開範囲を確定させてから、広いほうで全体を点検する。

初期状態は全項目「未確認」。特定システムの監査結果ではない。
