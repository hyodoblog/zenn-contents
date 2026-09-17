# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with content in this repository.

## このリポジトリについて

[Zenn](https://zenn.dev/hyodoblog) へ投稿する記事と本を管理する。GitHub 連携なので、
**`main` へ push した時点で Zenn へ反映される**。push は公開操作にあたる。

コードではなく文章のリポジトリ。したがってレビューの観点も、動くかどうかではなく
「読んだ人が同じ判断をできるか」になる。

## 記事を書く前に必ず読むもの

1. **[`writing-style.md`](./writing-style.md)** — 本人が書いた初期記事から整理した文体の特徴と執筆基準
2. **同ファイルの参照記事から 2〜3 本** — 今回の題材に近い初期記事を選ぶ。短い技術メモも基準にする

直近の記事は背景や関連する実装の確認に使う。文体は直近の記事だけから推定しない。
後年の改稿で語り口が変わった記事もあるため、必要に応じて Git 履歴を確認する。
本人の最新の指示を優先し、記事ごとに構成と長さを選ぶ。

## コマンド

```bash
pnpm install
pnpm dev                    # zenn preview（http://localhost:3000）
npx zenn new:article        # 記事を新規作成（slug は自動生成）
npx zenn list:articles      # 記事一覧
npx zenn list:books         # 本一覧
```

`README.md` に書かれている `npm run new:article` / `npm run preview` / `npm run textlint`
は **package.json に存在しない**。上のコマンドを使う。

### textlint について

`.textlintrc.json`（`preset-ja-technical-writing`）はあるが、**textlint 本体は
インストールされていない**。`pnpm dlx` 経由でもプリセットを解決できない。

文章の確認は `writing-style.md` の「書いた後の確認」を基準にする。
誤字、読みにくい長文、重複は見直すが、「！」「？」「と思います」などを一律に禁止しない。
lint を導入・実行する場合も、指摘をすべて直すために本人の語り口を消さない。

## ディレクトリ

| パス | 中身 |
| --- | --- |
| `articles/<slug>.md` | 記事。1 ファイル 1 記事 |
| `books/<slug>/` | 本。`config.yaml` と各チャプター |
| `images/<name>.png` | 記事から `/images/<name>.png` で参照する |
| `writing-style.md` | 文体メモ。記事を書く前に読む |

古い記事は Zenn のアップローダ（`storage.googleapis.com/zenn-user-upload/...`）を
直接参照している。新しい記事は `images/` に置いて相対パスで参照する。

## slug

zenn-cli の検証と同じ。

```
/^[0-9a-z\-_]{12,50}$/
```

小文字の英数字・ハイフン・アンダースコアで 12〜50 字。既存の記事はすべて 14 字の
英数字になっている。手で作るときも既存に合わせる。

## frontmatter

```yaml
---
title: "<紹介する内容が伝わるタイトル>"
emoji: "🐢"
type: "tech" # tech: 技術記事 / idea: アイデア
publication_name: "yoshinani_dev"
topics: ["nextjs", "supabase", "prisma", "vercel", "パフォーマンス"]
published: false
---
```

- `publication_name` は常に `yoshinani_dev`
- `topics` は 4〜5 個
- 詳しくは `writing-style.md`

## 記事を書くときの決めごと

1. **`published: false` で渡す。** 公開の判断は本人がする。AI が `true` にしない
2. **勝手に commit / push しない。** push は公開操作にあたる
3. **クライアント案件は固有名詞を伏せる。** プロダクト名・業務用語・アカウント ID を
   一般名に置き換え、冒頭の `:::message` で置き換えたことを断る
4. **裏を取る。** 記憶で書かない。型定義・公式ドキュメント・実際の出力を引用する
5. **測っていないことは書かない。** 「速くなったはず」を「速くなった」と書かない。
   数えた数字と測った数字を区別する
6. **本人の経験の範囲で書く。** 判断に関係する失敗は残すが、失敗談や教訓を必須にしない。
   体験・感情・採用理由を創作せず、手順紹介だけで完結してもよい

## 既知のずれ

- `README.md` の Usage 節が実態と合っていない（存在しない npm script を案内している）
- `README.md` の末尾に別リポジトリ（azukiazusa）向けの文章が残っている
- zenn-cli が 0.2.11 で、最新（0.5.x）から離れている

いずれも本人が把握したうえで放置している可能性がある。直すなら先に確認する。
