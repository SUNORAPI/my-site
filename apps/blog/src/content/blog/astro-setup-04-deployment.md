---
title: 'Astroで個人サイトを作る #4 GitHubとCloudflare Pagesでデプロイ'
description: 'SSH キー設定から GitHub への push、Cloudflare Pages へのデプロイまでを扱う。ホスティング比較と CI/CD の組み立て方も整理する。'
pubDate: 'Apr 17 2026'
series: 'Astroで個人サイトを作る'
seriesOrder: 4
---

> 本記事は単一プロジェクト構成の連載の 4 話目である。複数サイトを 1 リポジトリで運用したい場合は「Astroモノレポ構成編」を参照のこと。

前回までで Astro サイトの中身は一通り整った。今回は完成したサイトを公開する工程を扱う。GitHub にコードを置き、Cloudflare Pages から自動デプロイする流れを最短ルートで追う。

## GitHub リポジトリの作成

ブラウザで [github.com/new](https://github.com/new) を開き、次のように設定する。

- **Repository name**: `my-site`
- **Public / Private**: どちらでもよい（Cloudflare Pages は両方に対応）
- **Add a README**: チェックしない（後で衝突する）

「Create repository」をクリックする。空のリポジトリが作られる。

## SSH キーの準備

push 認証は SSH キー方式が一番楽である。一度設定すればパスワードや Personal Access Token を毎回入れる必要がない。

```sh
ssh-keygen -t ed25519 -C "your_email@example.com"
```

「Enter file in which to save the key」「Enter passphrase」と聞かれるが、いずれも空のまま Enter で構わない。

公開鍵を表示してコピーする。

```sh
$ cat ~/.ssh/id_ed25519.pub
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... your_email@example.com
```

GitHub の Settings → SSH and GPG keys → New SSH key を開き、コピーした内容を貼り付けて保存する。

接続テストする。

```sh
$ ssh -T git@github.com
The authenticity of host 'github.com (140.82.xxx.xxx)' can't be established.
ED25519 key fingerprint is SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'github.com' (ED25519) to the list of known hosts.
Hi <your-username>! You've successfully authenticated, but GitHub does not provide shell access.
```

初回のみフィンガープリント確認が入る。`yes` で進めて問題ない。

## ローカルから GitHub に push

`my-site/` 内で実行する。Astro CLI が初期化時に git init まで済ませているので、リモートを足して push するだけで済む。

```sh
cd my-site
git remote add origin git@github.com:<your-username>/my-site.git
git branch -M main
git push -u origin main
```

成功すると次のような出力になる。

```text
Enumerating objects: 32, done.
Counting objects: 100% (32/32), done.
Delta compression using up to 10 threads
Compressing objects: 100% (28/28), done.
Writing objects: 100% (32/32), 145.21 KiB | 1.45 MiB/s, done.
Total 32 (delta 1), reused 0 (delta 0), pack-reused 0 (from 0)
To github.com:<your-username>/my-site.git
 * [new branch]      main -> main
branch 'main' set up to track 'origin/main'.
```

## Cloudflare Pages へ接続

Cloudflare ダッシュボードで Workers & Pages → Pages → Create → Connect to Git の順に進む。初回は GitHub アプリのインストールが入り、対象リポジトリへの読み取りアクセスを許可する。

`my-site` を選び、次のビルド設定を入力する。

| 項目 | 設定値 |
| --- | --- |
| Production branch | `main` |
| Framework preset | Astro |
| Build command | `npm run build` |
| Build output directory | `dist` |
| Root directory | （空欄） |

「Save and Deploy」を押すと初回ビルドが始まる。ログには次のような表示が流れる。

```text
Cloning repository...
Initialized empty Git repository
...
Installing dependencies
Detected the following tools from environment: nodejs@22.12.0, npm@10.9.0
Installing project dependencies: npm clean-install --progress=false
...
Executing user command: npm run build
> astro build
...
[build] 8 page(s) built in 1.18s
[build] Complete!
Finished
Success: Assets published!
Deployment complete!
```

最後に次のような表示でデプロイ完了になる。

```text
Success! Your project is deployed to Region: Earth
You can preview your project at https://my-site-xxx.pages.dev
```

`https://my-site-xxx.pages.dev` の `xxx` 部分はランダムな英数字で、プロジェクトごとに自動付与される。これ以降は `main` ブランチに push するたびに本番デプロイ、それ以外のブランチには「プレビュー URL」が自動で割り振られる。

## 主要ホスティングの比較

参考までに主要な静的ホスティングを並べると次の通り。

- **Cloudflare Pages**: グローバル CDN の Cloudflare ネットワーク上で配信。帯域無制限、ビルド回数の枠も寛容で、無料枠の限界が大きい。Workers と統合しやすい。
- **Vercel**: Next.js のメーカーが運営。プレビューデプロイの UX が滑らかで、PR ごとに固有 URL が払い出される。Astro 向けのアダプタも公式に揃っている。
- **Netlify**: 老舗。フォーム、A/B、Edge Function、Identity などプラットフォーム機能が広い。
- **GitHub Pages**: GitHub 上のリポジトリから直接配信。最もシンプルで、外部サービスの登録が要らない。ビルド時間や成果物サイズに制限がある。

ブログを「コミットしたら自動で公開され続ける」状態にするだけなら、上記のどれを選んでも実現できる。判断軸を一つだけ挙げるなら、「将来エッジ関数を足したくなる可能性」だろう。

## GitHub Actions で自前 CI を組む場合

Cloudflare Pages の Git 連携デプロイは便利だが、ビルド前にテスト・lint を強制したい場合は GitHub Actions を使うのも一案である。

```yaml
name: deploy
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22.12.0
          cache: npm
      - run: npm ci
      - run: npx astro check
      - run: npm run build
      - uses: cloudflare/pages-action@v1
        with:
          apiToken: ${{ secrets.CF_API_TOKEN }}
          accountId: ${{ secrets.CF_ACCOUNT_ID }}
          projectName: my-site
          directory: dist
```

ポイントは三点。`npm ci` で lockfile を尊重したインストールを行い、ローカルと依存ツリーを揃える。`npx astro check` を CI に組み込み、`.astro` ファイルや frontmatter スキーマの違反を本番に出さない。API トークンと Account ID は GitHub Secrets に置き、コードに直接書かない。

## 環境変数とシークレットの扱い

Astro は `.env` ファイルを読む。クライアントに露出してよい変数は `PUBLIC_` プレフィックスを付ける。たとえば `PUBLIC_GA_ID=...` のように書くとブラウザ側のコードからも `import.meta.env.PUBLIC_GA_ID` で参照できる。

ホスティング側で設定する環境変数も同じ命名規約に従う。Cloudflare Pages なら Settings → Variables and Secrets で本番とプレビューを分けて設定できる。プレビュー側だけ別の Analytics ID を入れる、といった運用が定番である。

## 次回に向けて

ここまでで、サイトをインターネット上に出すところまでたどり着いた。最後の回では独自ドメインの取得と DNS 設定、HTTPS 化、リダイレクト戦略まで含めて「自分の住所を持つ」工程を扱う。
