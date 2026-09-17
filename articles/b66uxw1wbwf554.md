---
title: "pnpmからbunへ移行したときに、Vercelのビルドで詰まったこと"
emoji: "🍞"
type: "tech" # tech: 技術記事 / idea: アイデア
publication_name: "yoshinani_dev"
topics: ["bun", "pnpm", "vercel", "turborepo", "monorepo"]
published: true
---

Turborepo のモノレポを pnpm から bun へ移行しました。ローカルでは 3691 件のテストと型チェックが通りましたが、Vercel の Preview で Stripe SDK の型エラーが出ました。

調べると、ビルド環境の bun 1.3.14 が、手元の bun 1.4.0 で作ったロックファイルを読み込めていませんでした。今回は、その対応と、同じ移行で見つかった依存関係の設定漏れを紹介します。

:::message
本記事は業務システムの開発に伴走した記録です。クライアント名・プロダクト名・業務固有の用語は伏せ、アプリ名やプロジェクト名は一般的な名前に置き換えています。
:::

## つくっていたもの

移行の対象はこういう構成のモノレポです。

- **モノレポ**: Turborepo。`apps` 5 つ（管理画面・Bot 3 つ・ドキュメントサイト）、`packages` 3 つ
- **アプリ**: Next.js 16（App Router）
- **デプロイ**: 管理画面と Bot 2 つとドキュメントサイトが Vercel、Bot 1 つが Cloud Run
- **ランタイム**: Node 24。bun はパッケージマネージャとしてのみ使い、実行は Node のまま
- **移行前**: pnpm 11
- **移行後**: bun 1.4.0

移行時に、次の設定も変更しています。

| 項目 | 場所 | 理由 |
| --- | --- | --- |
| `trustedDependencies: []` | `package.json` | bun は既定で一部パッケージの install script を暗黙に許可する。空配列で打ち消す |
| `minimumReleaseAge = 86400` | `bunfig.toml` | pnpm は「分」、bun は「秒」。単位が違う |
| `turbo` を 2.10 以降へ | `package.json` | 後述しますが、これが後で効いてきます |

この状態で、Vercel のビルドも確認しました。

## Vercel の Preview で型エラーが出た

Vercel の Preview を CLI から流しました。ダッシュボードの設定に依存しない形で確認したかったからです。

```bash
vercel deploy --scope <team> --yes
```

Bot の 1 つが落ちました。

```
../../packages/features/src/stripe/client.ts(108,7): error TS2322:
  Type '"2026-05-27.dahlia"' is not assignable to type '"2026-08-26.dahlia"'.
Failed to type check.
Error: Command "turbo run build" exited with 1
```

Stripe SDK の `apiVersion` の型が合っていません。同種のエラーが合計 6 件出ていました。

しかしローカルでは通ります。

```bash
bun run turbo check-types --filter=web   # グリーン
bun run test --run                       # 321 ファイル / 3691 テスト全パス
```

CI も緑です。

Stripe のバージョンが変わっていないか確認しました。`bun.lock` を見ると `stripe@22.2.2` で固定されています。

```
"stripe": ["stripe@22.2.2", "", ...],
```

`package.json` の指定は `^22.2.2` です。ロックファイルが効いていれば 22.2.2 が入るはずで、実際ローカルではそうなっています。

## installCommand の設定を確認する

先に、心当たりを 1 つ持っていました。

このモノレポは各アプリの `vercel.json` に `"installCommand": "bun install"` を書いています。一方で、落ちたプロジェクトだけダッシュボード側に **`pnpm install` の上書きが残っていました**。

```bash
vercel project inspect <project> --scope <team>
```

```
Root Directory		apps/line
Framework Preset	Next.js
Build Command		turbo run build
Install Command		pnpm install      ← これ
```

この設定が原因だと思ったのですが、ビルドログを最後まで読むと、実際に走っていたのは `bun install` です。

```
Running "install" command: `bun install`...
```

**`vercel.json` の設定はダッシュボードの上書きより優先されます。** 疑っていた場所は、そもそも無効化されていました。

ここで 10 分ほど無駄にしています。エラーの近くだけ読んで、install ステップまで遡らなかったのが原因です。

## ビルド環境の bun が古かった

ビルドログを先頭から読み直しました。install の直後に、こう出ていました。

```
Running "install" command: `bun install`...
bun install v1.3.14 (0d9b296a)
  "lockfileVersion": 2,
^
error: Unknown lockfile version
at bun.lock:2:22
UnknownLockfileVersion: failed to parse lockfile: 'bun.lock'

warn: Ignoring lockfile
...
Saved lockfile
2666 packages installed [33.47s]
```

`warn: Ignoring lockfile` とあり、ロックファイルが使われていないことが分かりました。

bun 1.4 が書く `bun.lock` は `lockfileVersion: 2` です。Vercel のビルドイメージに載っている bun は 1.3.14 で、この形式を知りません。知らないので**エラーで止まらず、警告を出して `package.json` のレンジから全部解決し直します**。

`package.json` の `"packageManager": "bun@1.4.0"` は参照されていませんでした。

決め手になったのは末尾の数字です。修正後の正常なビルドと並べるとこうなります。

| | bun | packages installed |
| --- | --- | --- |
| 修正前 | 1.3.14 | **2666** |
| 修正後 | 1.4.0 | **1344** |

倍近く違います。ロックファイルが効いていない状態では、依存ツリーが別物になっていました。

## ビルド環境に bun 1.4.0 を入れる

方針は決まりました。Vercel 側の bun を 1.4 にすればいい。

まず npm で入れようとしました。

```json
{
  "installCommand": "npm i -g bun@1.4.0 && bun install --frozen-lockfile"
}
```

落ちました。ログはこうです。

```
added 2 packages in 2s
npm warn allow-scripts 1 package has install scripts not yet covered by allowScripts:
npm warn allow-scripts   bun@1.4.0 (postinstall: node install.js)
...
bun install v1.3.14 (0d9b296a)
error: Unknown lockfile version
warn: Ignoring lockfile
error: lockfile had changes, but lockfile is frozen
```

bun の npm パッケージは、postinstall でバイナリ本体をダウンロードします。**Vercel の npm はその postinstall をブロックします。** 結果、実体が入らず PATH 上の `/bun1/bun`（1.3.14）が使われ続けました。

ただし収穫はありました。`--frozen-lockfile` を付けたので、今度は `Ignoring lockfile` のあとに `error: lockfile had changes, but lockfile is frozen` が出て**ビルドが止まっています**。黙って別バージョンで成功するより、はるかに安全です。

最終形はこうなりました。公式インストーラで入れて、絶対パスで叩きます。

```json
{
  "installCommand": "curl -fsSL https://bun.sh/install | bash -s bun-v1.4.0 && $HOME/.bun/bin/bun install --frozen-lockfile"
}
```

`$HOME/.bun/bin/bun` と絶対パスで指定するのが要点です。PATH 任せにすると `/bun1/bun` に食われます。

結果です。

```
Running "install" command: `curl -fsSL https://bun.sh/install | bash -s bun-v1.4.0 && $HOME/.bun/bin/bun install --frozen-lockfile`...
bun install v1.4.0 (34cbb9a40)
1344 packages installed [26.90s]
...
Build Completed in /vercel/output [50s]
```

`Unknown lockfile version` も `Ignoring lockfile` も消え、Stripe の型エラーも一緒に消えました。

## アプリ側で宣言していなかった依存を追加する

ロックファイルが効くようになったら、今度は管理画面が別の理由で落ちました。

```
bun install v1.4.0 (34cbb9a40)
1344 packages installed [32.95s]      ← install は成功している
...
web:build: Error: Turbopack build failed with 5 errors:
web:build: Error: Cannot find module '@tailwindcss/postcss'
```

管理画面の `postcss.config.mjs` は、共有パッケージの設定を再エクスポートしているだけです。

```js
export { default } from "@repo/ui/postcss.config"
```

PostCSS のプラグイン名は**プロジェクトディレクトリ基点で解決されます**。つまり `apps/web` から `@tailwindcss/postcss` が引ける必要があります。ところが `apps/web/package.json` はこれを宣言しておらず、共有パッケージの依存がルートへホイストされていることに暗黙で依存していました。

pnpm では解決できていました。bun では解決できません。

面白いのは、同じ共有設定を使っているドキュメントサイトが**通っていた**ことです。理由は単純で、そちらは `devDependencies` に自分で `@tailwindcss/postcss` を書いていました。依存を宣言しているかどうかで、ビルド結果が分かれていました。

直し方は宣言を足すだけでした。`bun.lock` の差分は参照が 2 行増えるだけで、解決バージョンは変わりません。

## 別のマシンでも依存を入れ直す

ここまでで Vercel の 5 プロジェクトは全部通りました。残るは Cloud Run の Bot です。

こちらは `turbo prune` でサブセットを作ってから Docker イメージを焼きます。手元では通っていました。しかし別のマシンで実行したところ、こうなりました。

```
• turbo 2.9.18
 WARNING  An issue occurred while attempting to parse bun.lock:
   × Could not resolve workspaces.
  ╰─▶ Unsupported bun lockfile version: 2

  × Cannot prune without parsed lockfile.
```

またしても `lockfileVersion: 2` です。

移行 PR では `turbo` を `^2.9.18` から `^2.10.12` へ上げてあります。turbo 2.9 系は `lockfileVersion: 2` を解釈できず、`turbo prune` が失敗するからです。ここは事前に手当てされていました。

落ちたのは、そのマシンの `node_modules` が turbo を上げる前の状態だったからです。`bun install` をやり直すだけで直りました。

別のマシンでも、更新後の依存をインストールし直す必要がありました。

## ロックファイルが「無視される」ということ

今回確認した、ロックファイルを読めない場合の挙動を並べます。

| ツール | `lockfileVersion: 2` を読めないとき |
| --- | --- |
| bun 1.3.14（`bun install`） | 警告だけ出して**レンジから解決し直す** |
| bun 1.3.14（`--frozen-lockfile`） | エラーで止まる |
| turbo 2.9.18（`turbo prune`） | エラーで止まる |

止まってくれるものは気づけます。厄介なのは最初の行です。ビルドは成功しますし、多くの場合アプリも動きます。今回は Stripe SDK がたまたま型を変えていたので露見しました。型エラーの原因を調べるときも、先に install のログを確認しておけばよかったです。

今回の CI では、このバージョンの違いに気づけませんでした。GitHub Actions は `oven-sh/setup-bun` でバージョンを指定しているので、常に 1.4.0 で走ります。CI とデプロイ先で、同じ名前の別バージョンが動いていました。

## おわりに

bun への移行そのものは、体感としては良い変更でした。インストールは速く、設定は `bunfig.toml` と `package.json` に収まっています。ロックファイルも pnpm の 17122 行から 3662 行へ減りました。

一方で、bun はまだホスティング側の対応が追いついていない場面があります。今回引っかかった `lockfileVersion: 2` は、Vercel のビルドイメージが bun 1.4 以降になれば消える問題です。それまでは `installCommand` で自前のバージョンを入れる形が要ります。

暫定の対処なので、リポジトリの規約に「Vercel のイメージが bun 1.4 以降になったら素の `bun install --frozen-lockfile` へ戻してよい」と条件つきで書き残しました。

## 参考

- [bun install — Bun Docs](https://bun.sh/docs/cli/install)
- [Lockfile — Bun Docs](https://bun.sh/docs/install/lockfile)
- [bunfig.toml — Bun Docs](https://bun.sh/docs/runtime/bunfig)
- [Configuring a build — Vercel Docs](https://vercel.com/docs/deployments/configure-a-build)
- [Project Configuration: installCommand — Vercel Docs](https://vercel.com/docs/project-configuration#installcommand)
- [turbo prune — Turborepo Docs](https://turborepo.com/docs/reference/prune)
