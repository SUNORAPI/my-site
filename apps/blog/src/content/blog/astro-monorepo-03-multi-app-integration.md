---
title: 'Astroモノレポ構成編 #3 複数アプリの作成'
description: 'apps/ 配下に blog / tech / app / wiki の 4 つの Astro アプリを生成し、pnpm install と pnpm approve-builds で依存を整える。'
pubDate: 'Apr 29 2026'
---

> 本記事は最初からモノレポ構成でセットアップする連載の 3 話目である。1 リポジトリで 1 サイトだけ運用したい場合は「Astroで個人サイトを作る」連載を参照のこと。

前回までで `pnpm-workspace.yaml` を含むワークスペースの土台ができた。今回は `apps/` 配下に 4 つの Astro アプリを生成し、開発サーバが立ち上がるところまで進める。

## アプリの役割

ここで作る 4 アプリは次のように使い分けることを想定している。

- **`apps/blog`**: 個人ブログ。雑記中心。Blog テンプレートで生成する。
- **`apps/tech`**: 技術メモ・プロジェクト紹介。Minimal テンプレートで生成し、後から拡張する。
- **`apps/app`**: Web アプリ本体。動的処理が入る想定。Minimal テンプレート。
- **`apps/wiki`**: 攻略・Wiki 系。データ量が多くなる想定。Minimal テンプレート。

ここでは「個人ブログ」と「技術ブログ」を意図的に分離する。読者層が違う場合（前者は友人・趣味仲間、後者はエンジニア・採用担当）、サブドメインで分けたほうが SEO 上もブランディング上も整理しやすい。共通レイアウトは後から `packages/ui` に切り出すので、デザインの統一感は保てる。

## アプリの生成

`apps/` ディレクトリに入って Astro CLI を実行する。

```sh
cd apps
pnpm create astro blog -- --template blog
pnpm create astro tech -- --template minimal
pnpm create astro app -- --template minimal
pnpm create astro wiki -- --template minimal
```

`--template` フラグでテンプレートを直接指定するので、対話プロンプトを最小限で済ませられる。途中で「Do you want to install dependencies?」と聞かれるが、ここでは No でよい（後でルートから一括インストールする）。「Initialize a new git repository?」も No にする（親ディレクトリで Git を管理するため）。

生成後の構造はこうなる。

```sh
$ ls -R apps
apps:
app  blog  tech  wiki

apps/app:
README.md  astro.config.mjs  package.json  public  src  tsconfig.json
...
```

## ルートで一括インストール

ルートに戻り、`pnpm install` を 1 回叩く。これで 4 アプリすべての依存が解決される。

```sh
$ cd ..
$ pnpm install
Scope: all 4 workspace projects
Packages: +315
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Progress: resolved 388, reused 1, downloaded 316, added 315, done

╭ Warning ───────────────────────────────────────────────────────────────────────╮
│                                                                                │
│   Ignored build scripts: esbuild, sharp.                                       │
│   Run "pnpm approve-builds" to pick which dependencies should be allowed to    │
│   run scripts.                                                                 │
│                                                                                │
╰────────────────────────────────────────────────────────────────────────────────╯
Done in 3.9s using pnpm v10.33.0
```

「Scope: all 4 workspace projects」と出ているのが、ワークスペースが正しく認識されている証拠である。

## ビルドスクリプトの承認

警告に出ている `esbuild` と `sharp` は、画像処理やバンドリングに使われるネイティブモジュールで、インストール時に追加のセットアップスクリプトを実行する必要がある。pnpm v7 以降はセキュリティ上の理由で、これらを明示的に承認する仕組みになっている。

```sh
$ pnpm approve-builds
? Choose which packages to build (Press <space> to select, <a> to toggle all, <i> to invert selection) …
❯ ○ esbuild
  ○ sharp
```

`a` を押して全選択（`●`）にしてから Enter で確定する。これで Astro が画像最適化に使う `sharp` と、ビルドに使う `esbuild` が正しく動作するようになる。

承認内容は `package.json` の `pnpm.onlyBuiltDependencies` フィールドに記録され、次回以降の `pnpm install` では自動で適用される。

## 各アプリの `package.json` を確認

`pnpm --filter <name>` で指定するのはディレクトリ名ではなく `package.json` の `name` フィールドである。Astro CLI で `pnpm create astro blog` のように作った場合、`name` は自動で `blog` になる。ディレクトリ名と一致させておくとフィルタが直感的なので、次のように確認しておく。

```sh
$ grep '"name"' apps/*/package.json
apps/app/package.json:  "name": "app",
apps/blog/package.json: "name": "blog",
apps/tech/package.json: "name": "tech",
apps/wiki/package.json: "name": "wiki",
```

スコープを付けたい場合は `@my/blog` のようにすることもできるが、本連載ではスコープ無しで進める。

## ポートの衝突を避ける

Astro の開発サーバはデフォルトで `4321` ポートに立ち上がる。何もしないと「すでに使われている」と判断されて自動的に `4322`、`4323` と空きを探す挙動になる。これはこれで動くが、URL が毎回ぶれるのは不便である。

各アプリの `astro.config.mjs` でポートを固定しておくと迷子にならない。

```js
// apps/blog/astro.config.mjs
import { defineConfig } from 'astro/config';

export default defineConfig({
  site: 'https://blog.example.com',
  server: { port: 4321 },
});
```

同様に tech → 4322、app → 4323、wiki → 4324 のように並べておく。

## 開発サーバを起動する

ルートから `pnpm --filter` で各アプリを呼び出せる。

```sh
$ pnpm --filter blog dev

> blog@0.0.1 dev /path/to/my-project-monorepo/apps/blog
> astro dev

[vite] connected.
17:21:15 [types] Generated 0ms
17:21:15 [content] Syncing content
17:21:16 [content] Synced content
17:21:16 [vite] Re-optimizing dependencies because vite config has changed
 astro  v6.1.8 ready in 690 ms

┃ Local    http://localhost:4321/
┃ Network  use --host to expose

17:21:16 watching for file changes...
```

ブラウザで `http://localhost:4321/` を開き、Blog テンプレートのページが表示されればワークスペース構成は成功している。`Ctrl + C` で停止すると、

```text
^C
ERR_PNPM_RECURSIVE_RUN_FIRST_FAIL  blog@0.0.1 dev: `astro dev`
Command failed with signal "SIGINT"
```

のようなメッセージが出るが、これは「ユーザーが Ctrl+C で止めた」ことを pnpm が報告しているだけで、エラーではない。

他のアプリも同様に起動できる。

```sh
pnpm --filter tech dev   # http://localhost:4322
pnpm --filter app dev    # http://localhost:4323
pnpm --filter wiki dev   # http://localhost:4324
```

`pnpm --filter "*" dev` のように全選択すると、4 アプリを別々のポートで一斉に立ち上げることもできる。

## ビルド確認

`pnpm --filter blog build` で `apps/blog/dist/` に静的ファイルが生成される。Blog テンプレートで作ったアプリのビルドログはおおむね次のようになる。

```sh
$ pnpm --filter blog build

> blog@0.0.1 build /path/to/my-project-monorepo/apps/blog
> astro check && astro build

08:55:23 [build] output: "static"
08:55:23 [build] directory: /path/to/my-project-monorepo/apps/blog/dist/
...
17:48:59 ▶ /_astro/blog-placeholder-3.B-yKgT8r_Z2srqQK.webp (before: 21kB, after: 14kB) (+21ms) (9/12)
17:48:59 ✓ Completed in 398ms.
17:48:59 [build] ✓ Completed in 784ms.
17:48:59 [@astrojs/sitemap] `sitemap-index.xml` created at `dist`
17:48:59 [build] 8 page(s) built in 1.18s
17:48:59 [build] Complete!
```

`apps/blog/dist/index.html` などが生成されていれば成功である。`ls -R apps/blog/dist` で中身を確認できる（フォルダパスを単体で打つと `zsh: permission denied` と返るが、これは「シェルが実行ファイルだと解釈した」だけで故障ではない）。

## VS Code Workspace（任意）

複数アプリを扱う日常では、エディタが「どのアプリのファイルを編集しているか」を見失いがちである。`.code-workspace` ファイルを作って各アプリを独立したフォルダとして扱わせると便利。

```jsonc
{
  "folders": [
    { "path": "apps/blog", "name": "blog" },
    { "path": "apps/tech", "name": "tech" },
    { "path": "apps/app", "name": "app" },
    { "path": "apps/wiki", "name": "wiki" },
    { "path": ".", "name": "(root)" }
  ]
}
```

## 次回に向けて

次回は、4 アプリに散らばっているヘッダ・フッタ・SEO まわりを `packages/ui` に切り出し、`workspace:*` プロトコルで各アプリから参照する手順を扱う。
