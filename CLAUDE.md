# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with content in this repository.

## このリポジトリについて

[Zenn](https://zenn.dev/hyodoblog) へ投稿する記事と本を管理する。GitHub 連携なので、
**`main` へ push した時点で Zenn へ反映される**。push は公開操作にあたる。

コードではなく文章のリポジトリ。したがってレビューの観点も、動くかどうかではなく
「読んだ人が同じ判断をできるか」になる。

## 記事を書く前に必ず読むもの

1. **[`writing-style.md`](./writing-style.md)** — 文体・構成・禁則。過去記事から抽出した型
2. **直近の記事を 2〜3 本** — `articles/` の更新が新しいもの。温度感を合わせるため

この 2 つを読まずに書き始めると、ほぼ確実に「報告書」になって差し戻される。

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

そのため、主要ルールは以下のスクリプトで機械的に確認する。

```bash
python3 - articles/<slug>.md <<'PY'
import re, pathlib, sys
lines = pathlib.Path(sys.argv[1]).read_text().split("\n")
body, in_code, in_fm = [], False, False
for i, l in enumerate(lines, 1):
    if l.startswith("---") and i < 10: in_fm = not in_fm; continue
    if in_fm: continue
    if l.startswith("```"): in_code = not in_code; continue
    if in_code or l.startswith(("|", "#", ":::")): continue
    body.append((i, l))
issues = 0
for i, l in body:
    for s in re.split(r"(?<=。)", l):
        t = re.sub(r"`[^`]*`|\[[^\]]*\]\([^)]*\)|\*\*", "", s).strip()
        if len(t) > 100: print(f"100字超 L{i} ({len(t)}): {t[:60]}"); issues += 1
        if s.count("、") >= 4: print(f"読点過多 L{i}: {s[:60]}"); issues += 1
    if re.search(r"(である。|でない。|だろう。)", l): print(f"である調 L{i}"); issues += 1
    for m in re.finditer(r"[一-鿿]{7,}", l): print(f"連続漢字 L{i}: {m.group()}"); issues += 1
    for w in ["かもしれま", "ではないでしょうか", "たいと思い"]:
        if w in l: print(f"弱い表現 L{i}: {w}"); issues += 1
print("issues:", issues)
PY
```

あわせて次も 0 件であること。

```bash
grep -c '[！？０-９]' articles/<slug>.md   # 感嘆符・疑問符・全角数字
```

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
title: "<現象>。<意外な真因や結論>"
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
6. **失敗を省略しない。** 詰まった過程と誤診が記事の本体になる

## 既知のずれ

- `README.md` の Usage 節が実態と合っていない（存在しない npm script を案内している）
- `README.md` の末尾に別リポジトリ（azukiazusa）向けの文章が残っている
- zenn-cli が 0.2.11 で、最新（0.5.x）から離れている

いずれも本人が把握したうえで放置している可能性がある。直すなら先に確認する。
