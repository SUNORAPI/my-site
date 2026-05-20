---
title: 'Astroモノレポ構成編 #2 ワークスペースの初期化'
description: 'pnpm init から pnpm-workspace.yaml の作成までを行い、ルートの責務を整える。'
pubDate: 'Apr 27 2026'
---

> 本記事は最初からモノレポ構成でセットアップする連載の 2 話目である。1 リポジトリで 1 サイトだけ運用したい場合は「Astroで個人サイトを作る」連載を参照のこと。

前回はモノレポを採用する目的と最終構成を確認した。今回はリポジトリの土台を作る。

## ルートディレクトリの作成

任意の場所にモノレポ用のディレクトリを作る。名前は何でもよいが、本連載では `my-project-monorepo` とする。

```sh
mkdir my-project-monorepo && cd my-project-monorepo
```

以降のコマンドは、断りがない限りすべてこのディレクトリ（リポジトリのルート）で実行する。

## `pnpm init` で package.json を作る

Corepack が有効になっていれば、`pnpm` コマンドはすぐに使える。最初の `pnpm` 実行時には、pnpm 本体のダウンロード確認が入る。

```sh
$ pnpm init
! Corepack is about to download https://registry.npmjs.org/pnpm/-/pnpm-10.33.0.tgz
? Do you want to continue? [Y/n] y
Wrote to /path/to/my-project-monorepo/package.json
{
  "name": "my-project-monorepo",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "packageManager": "pnpm@10.33.0"
}
```

`packageManager` フィールドに pnpm のバージョンが自動で書き込まれる。Corepack 対応環境であれば、このフィールドに従って pnpm のバージョンが固定されるので、別のマシンでチェックアウトしても同じバージョンが使われる。

確認しておく。

```sh
$ pnpm -v
10.33.0
```

## ルート `package.json` の整備

初期生成された `package.json` を、ワークスペースのルートとして使う形に整える。具体的には次のように書き換える。

```json
{
  "name": "my-project-monorepo",
  "private": true,
  "packageManager": "pnpm@10.33.0",
  "engines": {
    "node": ">=22.12.0"
  },
  "scripts": {
    "dev:blog": "pnpm --filter blog dev",
    "dev:tech": "pnpm --filter tech dev",
    "dev:app": "pnpm --filter app dev",
    "dev:wiki": "pnpm --filter wiki dev",
    "build": "pnpm -r build",
    "check": "pnpm -r exec astro check"
  }
}
```

ポイントは次の通り。

- `"private": true` を必ず付ける。ルートを誤って公開する事故を防ぐ。
- `packageManager` フィールドで pnpm のバージョンを固定する。Corepack を有効にしている環境ではこの宣言を尊重して自動で正しいバージョンを使う。
- `engines.node` で Node のバージョン要件を宣言する。

各アプリは次回追加するので、`scripts` 内の `dev:*` は受け皿として並べておくだけでよい。

## `pnpm-workspace.yaml` の作成

ルートに `pnpm-workspace.yaml` を作り、どのディレクトリをワークスペースとして扱うかを宣言する。

```sh
touch pnpm-workspace.yaml
```

中身は次のように書く。

```yaml
packages:
  - 'apps/*'
  - 'packages/*'
```

`apps/` と `packages/` ディレクトリはまだないが、glob にマッチするディレクトリがなければ単に無視される。次回からこの配下にアプリを追加していく。

## ディレクトリの土台を作る

`apps/` と `packages/` を空のまま用意しておく。

```sh
mkdir apps packages
```

この時点でのリポジトリ構造はこうなる。

```
my-project-monorepo/
├── apps/
├── packages/
├── package.json
└── pnpm-workspace.yaml
```

## `.gitignore` の準備

Git で管理しないファイルを最初に決めておく。`node_modules` や `dist` が誤ってコミットされると、後から消すのは面倒なので最初に整える。

```sh
cat << EOF > .gitignore
node_modules/
dist/
.astro/
.DS_Store
*.log
EOF
```

pnpm の場合、`node_modules` は各ワークスペースに加えてルート直下にも作られる。`.gitignore` は単一行で `node_modules/` と書いておけば、すべての階層で無視される。

## Git の初期化

リポジトリとして初期化しておく。

```sh
git init
```

リモートへの push は、アプリを足してビルドが通る状態になってから行う。先に push しても困ることはないが、本連載では「動く状態になってから push」の順で進める。

## `pnpm --filter` の使い方の予習

ワークスペースができてくると、特定アプリだけにコマンドを実行する場面が増える。`pnpm --filter` フラグを使う。

- `pnpm --filter blog dev`: `blog` パッケージで `dev` スクリプトを実行。
- `pnpm --filter blog add @astrojs/mdx`: `blog` に依存を追加。
- `pnpm -r build`: 全ワークスペースで `build` を実行（`-r` は recursive）。
- `pnpm --filter "...blog" build`: `blog` とその依存パッケージだけビルド。

特に最後の「依存つきフィルタ」は、後から共通パッケージを追加したときに役立つ。

なお、`--filter` で指定するのは「ディレクトリ名」ではなく、各パッケージの `package.json` の `name` フィールドである。アプリのディレクトリ名と `name` を一致させておくとフィルタが直感的になる。

## ルートに置くべき設定ファイル

複数アプリで共通の設定は、ルートに置くと管理が一箇所に集まる。次回以降アプリを追加する中で、必要に応じて足していく。

- **TypeScript**: `tsconfig.base.json` をルートに置き、各アプリの `tsconfig.json` から `extends` で参照する。
- **ESLint**: フラットコンフィグ形式の `eslint.config.js` をルートに置く。
- **Prettier**: `.prettierrc` と `.prettierignore` をルートに置く。
- **EditorConfig**: `.editorconfig` をルートに置く。

各アプリのテンプレートに同名のファイルが残っているとどちらが優先されるか曖昧になるので、ルートに集約したら原則アプリ配下からは削除する。

## 次回に向けて

ここまでで、ワークスペースの土台が整った。次回は `apps/` 配下に Astro アプリを 4 つ作成し、`pnpm install` でまとめて依存を解決する工程を扱う。`pnpm create astro` の出力や、`pnpm approve-builds` の挙動なども示しながら進める。
