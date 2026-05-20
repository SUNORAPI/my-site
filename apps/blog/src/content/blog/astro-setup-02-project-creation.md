---
title: 'Astroで個人サイトを作る #2 プロジェクト生成とディレクトリ構造'
description: 'npm create astro でテンプレートを生成し、生成物の意味とコンテンツコレクションを読み解く。'
pubDate: 'Apr 15 2026'
---

> 本記事は単一プロジェクト構成の連載の 2 話目である。複数サイトを 1 リポジトリで運用したい場合は「Astroモノレポ構成編」を参照のこと。

前回は Node を入れる下準備を終えた。今回は実際にプロジェクトを作り、生成物の意味を確認しながら Astro の基本的な約束ごとを押さえる。

## テンプレートからの生成

Astro プロジェクトは公式の対話式 CLI から生成する。

```sh
npm create astro@latest my-site
```

このコマンドを実行すると、いくつかの質問に答えるだけで雛形が立ち上がる。本連載では Blog テンプレートを起点にする方針なので、迷ったら次のように選ぶ。

| 質問 | 選択 |
| --- | --- |
| Where should we create your new project? | そのまま Enter（`./my-site`） |
| How would you like to start your new project? | Use blog template |
| Do you plan to write TypeScript? | No（後から有効化できる） |
| Install dependencies? | Yes |
| Initialize a new git repository? | Yes |

完了後の表示には次のような案内が出る。

```text
 added 480 packages, and audited 481 packages in 12s
 ...

 Liftoff confirmed. Explore your project!

 Enter your project directory using cd ./my-site
 Run npm run dev to start the dev server. CTRL+C to stop.
 ...
```

`my-site/` ディレクトリに移動して開発サーバを起動する。

```sh
cd my-site
npm run dev
```

ブラウザで `http://localhost:4321/` を開くと「Welcome to the official Astro blog starter template」というサンプルが見える。ここがスタート地点になる。

## 生成されるディレクトリの読み方

Blog テンプレートで生成されるディレクトリは大筋でこのような構造になる。

```
my-site/
├── public/
├── src/
│   ├── assets/
│   ├── components/
│   ├── content/
│   │   └── blog/
│   ├── layouts/
│   ├── pages/
│   │   ├── index.astro       /        トップページ
│   │   ├── about.astro       /about
│   │   └── blog/
│   │       ├── index.astro   /blog
│   │       └── [...slug].astro
│   ├── styles/
│   ├── consts.ts
│   └── content.config.ts
├── astro.config.mjs
├── package.json
└── tsconfig.json
```

`public/` はビルド時にそのまま `dist/` 直下にコピーされる素材を置く場所である。`favicon.svg` や `robots.txt` のようにパスを変更したくないファイルが対象になる。`src/assets/` はビルドパイプラインで最適化したい画像。`src/components/` は `.astro` コンポーネント、`src/layouts/` はページ全体のラッパーレイアウト、`src/pages/` がファイルベースルーティングの起点となる。

## ファイルベースルーティングの基本

Astro のルーティングは `src/pages/` 配下のファイルパスがそのまま URL に対応する。`src/pages/about.astro` は `/about`、`src/pages/blog/index.astro` は `/blog/` というふうに、推測しやすい構造になっている。

動的なルートは `[slug].astro` のように角括弧で表現し、`getStaticPaths` で静的に展開する。ブログの個別ページは `src/pages/blog/[...slug].astro` のように rest パラメータを使い、コンテンツコレクションから記事の slug を流し込むのが典型的なパターンとなる。

## コンテンツコレクションという仕組み

Blog テンプレートで生成される `src/content.config.ts` はおおよそ次のような形になっている。

```ts
import { defineCollection, z } from 'astro:content';
import { glob } from 'astro/loaders';

const blog = defineCollection({
  loader: glob({ base: './src/content/blog', pattern: '**/*.{md,mdx}' }),
  schema: ({ image }) =>
    z.object({
      title: z.string(),
      description: z.string(),
      pubDate: z.coerce.date(),
      updatedDate: z.coerce.date().optional(),
      heroImage: z.optional(image()),
    }),
});

export const collections = { blog };
```

ここで宣言したスキーマに違反する frontmatter を書いた記事は、ビルド時にエラーになる。`pubDate` を書き忘れたまま push しても CI が落ちるので、メタデータの抜け漏れを早い段階で発見できる。`z.coerce.date()` のように文字列から `Date` への変換も担ってくれるので、テンプレート側では `pubDate.toISOString()` のようなメソッドを安心して呼べる。

## 記事を 1 本書いてみる

`src/content/blog/hello.md` を作って frontmatter を埋める。

```md
---
title: 'はじめての投稿'
description: 'Astro Blog テンプレートで最初の記事を書く。'
pubDate: '2026-04-15'
---

## 見出し

本文はそのまま Markdown が使える。
```

`npm run dev` を立ち上げたまま `/blog/hello/` にアクセスすると、即座に記事が表示される。Astro の HMR は Vite に支えられており、`.astro` ファイルだけでなく Markdown の変更も差分で即反映される。

## ビルドと preview

`npm run build` で静的ファイルが `dist/` に出力される。`npm run preview` は `dist/` をローカルでホストするコマンドで、本番に近い挙動を確認するのに使う。

```sh
$ npm run build
> astro build
...
[build] 8 page(s) built in 1.18s
[build] Complete!
```

ビルドが通れば、デプロイ可能な状態になっている。

## 次に向けて

ここまでで、テンプレートの中身とコンテンツコレクションの動きを掴めた。次回は雛形に手を入れて、ヘッダ・フッタ・SEO・OGP まわりを整える工程に進む。
