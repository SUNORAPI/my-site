---
title: 'Astroで個人サイトを作る #3 レイアウトとカスタマイズ'
description: 'Blog テンプレートを土台に、レイアウト・SEO・OGP・タイポグラフィを整える。デザイン判断の周辺知識も合わせて扱う。'
pubDate: 'Apr 16 2026'
---

> 本記事は単一プロジェクト構成の連載の 3 話目である。複数サイトを 1 リポジトリで運用したい場合は「Astroモノレポ構成編」を参照のこと。

前回は Blog テンプレートから生成された雛形を一通り読み解いた。今回はその雛形に手を入れて、サイトとしての顔を作っていく。

## レイアウトを「使い回す単位」として捉える

Astro におけるレイアウトは、ページの外枠を担当するコンポーネントである。`src/layouts/BaseLayout.astro` のような名前で作り、`<html>` から `<head>` の中身、ヘッダ、フッタ、メインスロットまでを面倒みる。Blog テンプレートには `BlogPost.astro` のような専用レイアウトも含まれており、ブログ記事用の見た目はこちらに集約されている。

レイアウトを設計するときは「ページごとにどこまで違っていいのか」を最初に決めると後がきれいになる。よくある分け方は次の通り。

- **共通**: `<head>` の title/description、フォント、グローバル CSS、ヘッダ、フッタ
- **ページ別に差し替えたい**: OGP 画像、構造化データ、`prev/next` ナビゲーション
- **ページ内コンテンツ固有**: 目次、シェアボタン、関連記事

「共通」を Base レイアウトに、「差し替え」を props として受け取り、ページ固有のものはページ側で書く、という三層構造に落とすと、変更したときの影響範囲が予測しやすい。

## トップページを書き換える

テンプレート初期状態の `src/pages/index.astro` には「Welcome to the official Astro blog starter template」というサンプル文言が入っている。本番公開する前にここを差し替えておきたい。極端な例だが、最小構成だけ示すなら次のような形になる。

```astro
---
import BaseLayout from '../layouts/BaseLayout.astro';
---
<BaseLayout title="My Site" description="個人サイト">
  <main>
    <h1>準備中</h1>
    <p>近日公開予定。</p>
  </main>
</BaseLayout>
```

サンプル文言のまま公開してしまうと検索エンジンや SNS のサムネイルに残るので、最初のデプロイ前に一度差し替えておくのが無難である。

## SEO の最低ラインを整える

Astro 単体で SEO が完結するわけではないが、最低限押さえるべきタグはほぼ決まっている。

- `<title>` と `<meta name="description">`
- canonical URL（`<link rel="canonical">`）
- OGP（`og:title`, `og:description`, `og:image`, `og:url`, `og:type`）
- Twitter Card（`twitter:card`, `twitter:image`）
- 構造化データ（JSON-LD）

これらを毎回手書きするのは大変なので、`<BaseHead>` のような専用コンポーネントを作り、props で必要な値を受け取って `<head>` を組み立てる方式がよく使われる。テンプレートにもこの形が含まれているはずなので、自分の用途に合わせて拡張していけばよい。

canonical URL は `Astro.site` と `Astro.url.pathname` を組み合わせて作る。`astro.config.mjs` の `site` プロパティを設定しておかないと `Astro.site` が `undefined` になり、サイトマップや RSS の URL も壊れるので、最初に独自ドメインを決めておくか、仮の値を入れておくと安全である。

```js
// astro.config.mjs
import { defineConfig } from 'astro/config';
import mdx from '@astrojs/mdx';
import sitemap from '@astrojs/sitemap';

export default defineConfig({
  site: 'https://example.com',
  integrations: [mdx(), sitemap()],
});
```

## OGP 画像をどう用意するか

OGP 画像は SNS で記事が共有されたときに表示されるサムネイルで、クリック率に直結する。手で毎回作ると追いつかなくなるため、自動生成の仕組みを最初に入れておくと運用が楽になる。選択肢は大きく三つある。

1. **静的に作っておく**: Figma などで作成し、記事ごとに `heroImage` フィールドにパスを入れる。Astro Blog テンプレートはこの方式を採用している。
2. **ビルド時に生成する**: `satori` などのライブラリで JSX から PNG を生成し、ビルド時に出力する。
3. **エッジで生成する**: Vercel OG Image のようにリクエストごとに生成する。

静的サイトとして配信したい場合は 1 か 2 が現実的で、コンテンツコレクションのスキーマと統合しやすい 2 を取ると、運用フローが大きく軽くなる。

## タイポグラフィとフォント供給

Astro 公式の `astro:assets` には font プロバイダ機能があり、ローカルや CDN にあるフォントをビルドパイプラインに乗せられる。Blog テンプレートでは `Atkinson` のような可読性に優れたフォントが含まれており、`astro.config.mjs` で次のように宣言される。

```js
import { defineConfig, fontProviders } from 'astro/config';

export default defineConfig({
  experimental: {
    fonts: [
      {
        provider: fontProviders.local(),
        name: 'Atkinson',
        cssVariable: '--font-atkinson',
        variants: [/* ... */],
      },
    ],
  },
});
```

ブラウザに余計な JS を送らずに済むのが Astro の強みなので、フォントもセルフホストして CSS で扱うのが性能的に有利である。Cloudflare Pages のような CDN 配信であれば、Brotli 圧縮された woff2 をそのまま吐き出せばよい。

## ダークモードの実装方針

ダークモードを入れるかどうかは趣味の問題でもあるが、入れるなら次の三方式から選ぶ。

- **CSS `prefers-color-scheme` のみ**: ユーザーの OS 設定に従って自動切り替え。JS 不要。
- **トグル + localStorage**: ユーザーが手動で切り替えられる。`<script>` を 1 つ `<head>` 内で同期実行し、FOUC を避ける。
- **OS 連動 + 手動オーバーライド**: 上記の合わせ技。

Astro はクライアント JS をデフォルトで送らないので、手動切り替えを入れるときは「最初の HTML レンダリングよりも前にテーマを決める」必要がある。`<head>` 内のインラインスクリプトで `document.documentElement.dataset.theme` を即時設定する、という小さなトリックがほぼ必須になる。

## マークダウンの拡張

Astro は内部で `remark` / `rehype` のパイプラインを抱えており、`astro.config.mjs` でプラグインを追加できる。

- `remark-gfm`: GitHub Flavored Markdown（表、タスクリスト、打ち消し線）
- `remark-toc`: 目次の自動生成
- `rehype-slug` + `rehype-autolink-headings`: 見出しに ID と自動アンカーリンク
- `rehype-pretty-code` / `shiki`: シンタックスハイライト

ブログとして長く運用するつもりなら、最初にこのあたりを揃えておくと「目次がほしい」「見出しに直接リンクしたい」と思った瞬間に追加でき、後から導入してビルドが壊れる事故を避けられる。

## 次回に向けて

ここまでで、サイトの見た目と SEO の骨格がそろった。次回は完成したサイトを GitHub に押し上げ、Cloudflare Pages でデプロイするところまでを扱う。
