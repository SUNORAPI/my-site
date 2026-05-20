---
title: 'Astroモノレポ構成編 #5 マルチサイトのデプロイ'
description: '4 アプリを GitHub に push し、Cloudflare Pages 上で個別プロジェクトとしてサブドメイン配信する。'
pubDate: 'May 03 2026'
series: 'Astroモノレポ構成編'
seriesOrder: 5
---

> 本記事は最初からモノレポ構成でセットアップする連載の最終話である。1 リポジトリで 1 サイトだけ運用したい場合は「Astroで個人サイトを作る」連載を参照のこと。

連載最終回は、モノレポからの実デプロイを扱う。4 アプリをそれぞれ別のサブドメインに配信し、ひとつの push で必要なアプリだけビルド・デプロイされる状態を目指す。

## GitHub への push

リポジトリのルートで Git を初期化し、リモートを設定する。SSH 接続を使うのが楽である。

```sh
# SSH キーがなければ作る
ssh-keygen -t ed25519 -C "your_email@example.com"
cat ~/.ssh/id_ed25519.pub   # 表示された内容を GitHub に登録
```

GitHub の Settings → SSH and GPG keys → New SSH key で公開鍵を登録した後、接続確認をする。

```sh
$ ssh -T git@github.com
Hi <your-username>! You've successfully authenticated, but GitHub does not provide shell access.
```

GitHub 側で空のリポジトリ（例: `my-project-monorepo`）を作り、ルートから push する。

```sh
git add .
git commit -m "initial commit: monorepo structure"
git branch -M main
git remote add origin git@github.com:<your-username>/my-project-monorepo.git
git push -u origin main
```

成功すると次のような出力になる。

```text
Enumerating objects: 66, done.
Counting objects: 100% (66/66), done.
Compressing objects: 100% (60/60), done.
Writing objects: 100% (66/66), 329.76 KiB | 1.96 MiB/s, done.
To github.com:<your-username>/my-project-monorepo.git
 * [new branch]      main -> main
branch 'main' set up to track 'origin/main'.
```

`apps/`、`packages/`、`pnpm-workspace.yaml`、`package.json` が GitHub に上がっていて、`node_modules/` が含まれていなければ正解である。

## 「1 プロジェクト 1 アプリ」の原則

マルチサイトのデプロイで最初に決めるのは、ホスティング側のプロジェクト粒度である。基本方針はシンプルで、

> ホスティングのプロジェクトは、Astro アプリと 1 対 1 に対応させる

のが分かりやすい。Cloudflare Pages であれば次の 4 プロジェクトを作る。

| Pages プロジェクト名 | 対応アプリ | 想定ドメイン |
| --- | --- | --- |
| my-blog | apps/blog | blog.example.com |
| my-tech | apps/tech | tech.example.com |
| my-app | apps/app | app.example.com |
| my-wiki | apps/wiki | wiki.example.com |

それぞれのプロジェクトに同じ GitHub リポジトリを接続するが、Build command と Build output directory を別々に設定する。

## Cloudflare Pages 側の設定

ダッシュボードで Workers & Pages → Create application → Pages → Connect to Git に進み、対象リポジトリを選ぶ。ビルド設定は次のように入力する。

| 項目 | 設定値 |
| --- | --- |
| Project name | `my-blog`（プロジェクトごとに変える） |
| Production branch | `main` |
| Framework preset | Astro |
| Build command | `pnpm --filter blog build` |
| Build output directory | `apps/blog/dist` |
| Root directory | `/`（空欄またはルート） |

ここで重要なのが **Root directory を `apps/blog` ではなく `/`（ルート）に設定すること** である。Cloudflare Pages のビルド環境はリポジトリのルートで pnpm を実行する必要があり、`pnpm-workspace.yaml` を見つけられないと workspace の解決に失敗するからである。Root を `apps/blog` にしてしまうと、共有パッケージ `@my/ui` をどこから引いていいのか分からず、ビルドが落ちる。

`tech`、`app`、`wiki` も同じ要領で 3 プロジェクト追加する。Build command と Output directory だけが変わる。

## 環境変数 PNPM_VERSION の設定

Cloudflare Pages のビルド環境では、pnpm のバージョンを環境変数で指定すると安定する。各プロジェクトの Settings → Variables and Secrets で次を追加する。

| Variable name | Value |
| --- | --- |
| PNPM_VERSION | `10.33.0` |
| NODE_VERSION | `22.12.0` |

これで「Cloudflare 側の pnpm がローカルと違う」事故を避けられる。`package.json` の `packageManager` フィールドを尊重させたい場合は `ENABLE_COREPACK_STRICT=1` を併用する手もある。

## 初回デプロイのログ

「Save and Deploy」を押すと初回ビルドが走る。ログには次のような表示が流れる。

```text
Cloning repository...
...
Detected the following tools from environment: nodejs@22.12.0, pnpm@10.33.0
Installing project dependencies: pnpm install --frozen-lockfile
...
Executing user command: pnpm --filter blog build
> blog@0.0.1 build /opt/buildhome/repo/apps/blog
> astro check && astro build
...
[build] 8 page(s) built in 1.18s
[@astrojs/sitemap] `sitemap-index.xml` created at `dist`
[build] Complete!
Finished
Success: Assets published!
Deployment complete!
```

`https://my-blog.pages.dev` のような URL でアクセスできるようになる。

## 共通パッケージへの参照を考慮する

`packages/ui` を参照している `apps/blog` をビルドするには、`packages/ui` も巻き込んでビルドする必要がある。pnpm の `--filter` は依存解決を理解しているので、デフォルトでも `packages/ui` は同じ workspace 上にあれば自動で含まれる。明示したい場合は三点リーダ記法を使う。

```sh
pnpm --filter "blog..." build
```

`blog...` は「`blog` とその依存先（つまり `packages/ui`）」を意味する。Cloudflare Pages の Build command でこの書き方を使っても問題ない。

## 差分ビルド: 変更されたアプリだけビルドする

4 アプリすべてのデプロイが毎コミット走るのは無駄である。Cloudflare Pages では「Build Watch Paths」設定がある。プロジェクトごとに監視するパスを指定すると、それ以外のディレクトリだけが変わったコミットではビルドをスキップする。

- `my-blog`: `apps/blog/**`, `packages/**`, `pnpm-lock.yaml`, `pnpm-workspace.yaml`, `package.json`
- `my-tech`: `apps/tech/**`, `packages/**`, `pnpm-lock.yaml`, `pnpm-workspace.yaml`, `package.json`
- 以下同様

`packages/**` を必ず含めるのが重要。共通パッケージを更新したら、それを使うアプリは再ビルドが要るからである。

## GitHub Actions で揃える場合

ホスティングの Git 連携機能ではなく、自前 CI でビルド・デプロイを統合管理したい場合は GitHub Actions の matrix が便利である。

```yaml
name: deploy
on:
  push:
    branches: [main]

jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      apps: ${{ steps.filter.outputs.changes }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 2
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            blog: ['apps/blog/**', 'packages/**']
            tech: ['apps/tech/**', 'packages/**']
            app:  ['apps/app/**',  'packages/**']
            wiki: ['apps/wiki/**', 'packages/**']

  deploy:
    needs: changes
    if: ${{ needs.changes.outputs.apps != '[]' }}
    runs-on: ubuntu-latest
    strategy:
      matrix:
        app: ${{ fromJSON(needs.changes.outputs.apps) }}
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with:
          version: 10.33.0
      - uses: actions/setup-node@v4
        with:
          node-version: 22.12.0
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm --filter ${{ matrix.app }}... build
      - uses: cloudflare/pages-action@v1
        with:
          apiToken: ${{ secrets.CF_API_TOKEN }}
          accountId: ${{ secrets.CF_ACCOUNT_ID }}
          projectName: my-${{ matrix.app }}
          directory: apps/${{ matrix.app }}/dist
```

## 独自ドメインの紐付け

各 Pages プロジェクトの Custom domains から、対応するサブドメインを追加する。

- my-blog → `blog.example.com`
- my-tech → `tech.example.com`
- my-app → `app.example.com`
- my-wiki → `wiki.example.com`

Cloudflare に DNS を集約している場合は、ボタン一つで DNS レコードと TLS 証明書が自動で発行される。数分以内に HTTPS で接続できるようになる。

## サブドメイン間のリンク

サブドメインで分割すると、相対パスでサイト間を行き来できない。共通のフッタやヘッダから他サイトにリンクするときは、絶対 URL で書く。

```astro
---
const techUrl = 'https://tech.example.com';
---
<a href={techUrl}>テックブログへ</a>
```

将来 URL が変わる可能性を考えるなら、環境変数化しておく。Astro の `import.meta.env.PUBLIC_*` を使えば、Cloudflare Pages 側で本番とプレビューを切り替えるのも容易である。

## モニタリング

公開した後は、最低限のモニタリングがあると安心である。

- **稼働監視**: UptimeRobot / Better Stack の無料枠で 5 分間隔チェック。
- **アクセス解析**: GA4、Plausible、Umami のいずれか。Plausible / Umami はセルフホスト可能で、Cookie 不要なため法規制の側面でも軽い。
- **Real User Monitoring**: Cloudflare Web Analytics は Pages との統合が容易。
- **エラー検知**: Sentry。Astro 用の SDK が揃っている。

## 連載のまとめ

ここまでで「`mkdir my-project-monorepo` から始めて、pnpm workspace に 4 つの Astro アプリを並べ、`packages/ui` で共通レイアウトを切り出し、Cloudflare Pages で 4 サブドメインに配信する」という一連の流れが揃った。最初からこの構成で立ち上げておくと、サイトを増やすたびに `apps/<name>/` を作って Pages プロジェクトを追加するだけで運用できる。

逆に「1 リポジトリで 1 サイトだけ」の構成で十分なケースもある。そちらは別連載「Astroで個人サイトを作る」を参照のこと。
