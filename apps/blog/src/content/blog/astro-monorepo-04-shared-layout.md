---
title: 'Astroモノレポ構成編 #4 共通レイアウトの切り出し'
description: 'packages/ui を作り、共通コンポーネントを workspace:* プロトコルで各アプリから参照する。'
pubDate: 'May 01 2026'
series: 'Astroモノレポ構成編'
seriesOrder: 4
---

> 本記事は最初からモノレポ構成でセットアップする連載の 4 話目である。1 リポジトリで 1 サイトだけ運用したい場合は「Astroで個人サイトを作る」連載を参照のこと。

前回までで `apps/blog` / `apps/tech` / `apps/app` / `apps/wiki` の 4 アプリが揃った。それぞれに似たヘッダ・フッタ・SEO 用コンポーネントがあると、変更の度に 4 箇所書き直すことになる。今回はその重複を `packages/ui` に切り出し、ワークスペース内パッケージとして参照する構成を作る。

## 共通化するべきものとそうでないもの

最初に「共通化する対象」を絞り込む。今回の 4 アプリでよく挙がるのは次のあたり。

- **共通化したい**
  - フッタ（コピーライト、SNS リンク、姉妹サイトへの相互リンク）
  - `<BaseHead>` の SEO/OGP 構造
  - サイトカード、ボタン、リンクスタイルなどの基本パーツ
- **共通化したくない**
  - 各サイト固有のヘッダ（ナビゲーション項目が違う）
  - サイトごとのテーマカラーやタイポグラフィの強弱

「揃っていてほしいもの」だけを共通化するのが基本方針になる。

## `packages/ui` の作成

`packages/ui` ディレクトリを作り、その配下に `package.json` を置く。Astro コンポーネントは普通の `.astro` ファイルとして配置できる。

```sh
mkdir -p packages/ui/src/components
cd packages/ui
pnpm init
```

`pnpm init` で生成される `package.json` を次の形に整える。

```json
{
  "name": "@my/ui",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "exports": {
    "./components/*": "./src/components/*",
    "./styles/*": "./src/styles/*"
  },
  "peerDependencies": {
    "astro": "^5.0.0"
  }
}
```

- `"private": true` を付けて npm に publish されないようにする。
- `exports` で公開するエントリポイントを宣言する。`./components/BaseHead.astro` のようなパスで各アプリから import できるようになる。
- Astro 本体は `peerDependencies` にする。`packages/ui` が独立した Astro バージョンを抱えないようにするためである。

## 共通コンポーネントを 1 つ書いてみる

`packages/ui/src/components/Footer.astro` を作る。

```astro
---
const year = new Date().getFullYear();
---
<footer>
  <small>&copy; {year} My Sites</small>
  <nav>
    <a href="https://blog.example.com/">blog</a>
    <a href="https://tech.example.com/">tech</a>
    <a href="https://app.example.com/">app</a>
    <a href="https://wiki.example.com/">wiki</a>
  </nav>
</footer>

<style>
  footer {
    display: flex;
    justify-content: space-between;
    padding: 1rem;
    border-top: 1px solid var(--color-border, #ddd);
  }
</style>
```

サブドメイン間の相互リンクは、相対パスでは届かないので絶対 URL で書く。後から環境変数化したくなったら、`import.meta.env.PUBLIC_BLOG_URL` のような形に差し替えられるようにしておくと拡張しやすい。

## アプリから共通パッケージを参照する

各アプリの `package.json` に `@my/ui` を依存として宣言する。例として `apps/blog/package.json` の `dependencies` に追記する。

```json
{
  "dependencies": {
    "@my/ui": "workspace:*"
  }
}
```

`workspace:*` という記法が pnpm のワークスペースプロトコルである。「同じワークスペースの `@my/ui` を参照する」という意味になる。ルートで `pnpm install` を再実行する。

```sh
$ pnpm install
Scope: all 5 workspace projects
Lockfile is up to date, resolution step is skipped
Already up to date
Done in 336ms using pnpm v10.33.0
```

`apps/blog/node_modules/@my/ui` が `packages/ui` へのシンボリックリンクとして張られる。

アプリ側の `.astro` ファイルから次のように import できる。

```astro
---
import Footer from '@my/ui/components/Footer.astro';
---

<html lang="ja">
  <body>
    <slot />
    <Footer />
  </body>
</html>
```

## CSS をどう共有するか

Astro コンポーネント内に書いた `<style>` は、コンポーネントごとにスコープされる。共通コンポーネント側に書いたスタイルは、自動的にスコープ化されて各アプリのページに紛れ込む。

一方で、サイト全体の色やフォント、リセット CSS のような「グローバルに当てたい」スタイルは、別途 CSS ファイルとして配置する。

```css
/* packages/ui/src/styles/global.css */
:root {
  --color-bg: #ffffff;
  --color-text: #1a1a1a;
  --color-accent: #2563eb;
  --color-border: #e5e7eb;
}

body {
  background: var(--color-bg);
  color: var(--color-text);
  font-family: system-ui, sans-serif;
}
```

各アプリのレイアウトで次のように import する。

```astro
---
import '@my/ui/styles/global.css';
---
```

## カラーやテーマ変数のサイト別カスタマイズ

CSS カスタムプロパティを使うと、共通コンポーネントを共有しつつ、各アプリでテーマ色だけ差し替えるという運用がやりやすい。

```css
/* apps/tech/src/styles/theme.css */
:root {
  --color-accent: #16a34a;  /* tech だけ緑系に */
}
```

`theme.css` は `global.css` の後に読み込む。順序の制御はレイアウト内の import 順で行う。

## ホットリロードはどう動くか

ワークスペース内のパッケージを参照していると、共通パッケージ側のファイルを編集したときに各アプリの dev サーバが反応するかが気になる。pnpm は `node_modules` をシンボリックリンクで構成しているので、`packages/ui/src/components/Footer.astro` を直接編集すると、各アプリの Vite はそのファイルが変わったことを検知して HMR を発火する。中継のビルド工程は不要である。

ただし `package.json` の `exports` を変えたときは Vite の再起動が要る。経験則として「`package.json` を触ったら一度 dev を立て直す」と覚えておくとよい。

## ありがちな落とし穴

- **`workspace:*` が `latest` に勝手に置き換わる**: 一部のツールが publish 時に `workspace:*` を実際のバージョンに置換するが、publish しないパッケージで挙動が崩れることがある。`private: true` を付けて publish 対象から外しておくのが安全。
- **`.astro` ファイルの型定義が効かない**: TypeScript が `.astro` を解決できないと赤線まみれになる。各アプリの `src/env.d.ts` に `<reference types="astro/client" />` が入っているか確認する。
- **`tsconfig` の `references` を使うべきか**: モノレポでよく出てくる選択肢だが、Astro のビルドは Vite ベースで、TS の `references` を必要としない。引っ張り出すと逆に複雑になる場面が多い。

## 次回に向けて

次回は連載の締めくくりとして、4 アプリをそれぞれのサブドメインに配信するデプロイ戦略を扱う。GitHub への push、Cloudflare Pages 側の設定、差分ビルドの仕組みまで踏み込む。
