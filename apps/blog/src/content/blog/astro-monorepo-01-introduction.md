---
title: 'Astroモノレポ構成編 #1 全体像と環境準備'
description: '複数の Astro サイトを 1 リポジトリで運用するための前提を整理する。pnpm workspace の選定理由と最終的なディレクトリ構成を示す。'
pubDate: 'Apr 25 2026'
---

> 本連載は「複数の Astro サイト（ブログ、技術メモ、Web アプリ、Wiki など）を 1 リポジトリで運用する」構成を、最初からモノレポとしてセットアップする手順を扱う。1 リポジトリで 1 サイトだけ運用したい場合は別連載「Astroで個人サイトを作る」を参照されたい。シンプルさが欲しいなら単一プロジェクト編、最初から複数サイトを見据えるなら本連載、というのが選び方の目安になる。

連載初回となる本記事では、なぜモノレポ構成にするのか、どのツールを使うのか、最終的にどんなディレクトリ構造を目指すのかを整理する。実作業は次回から入る。

## モノレポという言葉が指す範囲

モノレポ（monorepo）は「複数のプロジェクトを 1 つのリポジトリで管理する構成」を広く指す言葉である。歴史的には Google や Facebook の巨大な単一リポジトリが代表例として語られるが、現代の Web フロントエンド文脈で「モノレポ」と言うときは、もっと小さい規模を想定していることが多い。本連載で作るのは次のような構成である。

- 個人ブログ（雑記中心）
- 技術ブログ（プロジェクト紹介を兼ねる）
- Web アプリ本体（動的処理あり）
- 攻略 / Wiki 系サイト

これらをサブドメインごとに切り分け、共通のヘッダ・フッタ・SEO 用コンポーネントは 1 箇所にまとめて使い回す、という構成を目指す。

## モノレポにする利点

複数サイトをひとつのリポジトリに置くと、次のような利点が出てくる。

- **横断的な変更がアトミックになる**: 共通レイアウトに手を入れて全サイトを追従させるとき、1 つの PR で済む。
- **依存バージョンを揃えやすい**: Astro やプラグインのバージョンを各サイトで揃えやすい。pnpm の `overrides` を使えば、リポジトリ全体で特定パッケージのバージョンを強制することもできる。
- **共通コードを切り出しやすい**: ヘッダ、フッタ、SEO まわりのコンポーネントを `packages/ui` のような場所に置き、各アプリから参照できる。
- **ツーリングの集約**: ESLint、Prettier、TypeScript の設定をルートに置けるため、設定ファイルの重複が消える。
- **デプロイの独立性**: ホスティング側で「変更があったアプリだけビルド」を仕掛けると、ブログを更新したときに Wiki まで再ビルドされる無駄がなくなる。

## pnpm workspace を選ぶ理由

JS モノレポを構成するツールには複数の選択肢がある。

- **pnpm workspace**: pnpm 本体に組み込まれた機能。`pnpm-workspace.yaml` で対象を宣言する。
- **Yarn workspaces**: Yarn 1/2/Berry にそれぞれ存在。
- **npm workspaces**: npm 7 以降。最も素朴。
- **Turborepo**: pnpm/Yarn/npm の workspace に被せてタスクのキャッシュと並列実行を担う。
- **Nx**: タスクランナーに加え、ジェネレータや依存グラフ分析まで含む。

本連載では pnpm workspace を採用する。理由は次の三点である。

1. **ディスク使用量とインストール速度**: pnpm はグローバルストアからハードリンクで参照する仕組みなので、同じ依存を複数アプリで重複保存しない。
2. **厳格な依存解決**: `package.json` に書いた依存しか import できない（phantom dependency を防ぐ）。
3. **ワークスペースのサポート**: `pnpm-workspace.yaml` だけで複数パッケージを宣言でき、追加のメタフレームワークが不要。

タスクの並列実行が遅くなってきたら、上に Turborepo を被せて段階的に高速化する、というのが筋の良い拡張パスになる。

## Node と pnpm のインストール

モノレポ構成では Node 22 系 LTS と pnpm v10 を使う。Node のバージョン管理は fnm を使うのが手軽である。

```sh
# macOS
brew install fnm
echo 'eval "$(fnm env --use-on-cd)"' >> ~/.zshrc
source ~/.zshrc

# Node のインストール
fnm install --lts
fnm use lts-latest
```

確認すると次のような表示になる。

```sh
$ node -v
v22.12.0
$ fnm --version
fnm 1.39.0
```

pnpm は Corepack 経由で入れるのが手早い。Corepack は Node に同梱されているので追加ダウンロードは不要である。

```sh
corepack enable
```

これだけで `pnpm` コマンドが使えるようになる（権限エラーが出る場合は `sudo corepack enable`）。後ほど `pnpm init` の初回実行時に、`pnpm@10.x.x` のバイナリが自動でダウンロードされる仕組みになっている。

## 想定する最終構成

本連載で作るリポジトリは、最終的に次のような構造を目指す。

```
my-project-monorepo/
├── apps/
│   ├── blog/        # 個人ブログ（blog.example.com）
│   ├── tech/        # 技術メモ・プロジェクト紹介（tech.example.com）
│   ├── app/         # Web アプリ本体（app.example.com）
│   └── wiki/        # 攻略・Wiki 系（wiki.example.com）
├── packages/
│   ├── ui/          # 共通コンポーネント
│   └── config/      # 共通設定（任意）
├── pnpm-workspace.yaml
├── package.json
└── .gitignore
```

`apps/` 直下に各 Astro アプリ、共通コードが必要になったら `packages/` 配下に切り出していく、というシンプルな分け方である。

## サブドメインで分けるか、パスで分けるか

複数サイトを 1 ドメイン下に同居させる場合、次の 2 通りがある。

1. **サブドメイン分割**: `blog.example.com`, `tech.example.com`, `app.example.com`。ホスティング側のプロジェクト分割と素直に対応する。
2. **パス分割**: `example.com/blog/`, `example.com/tech/`, `example.com/app/`。SEO 上の集約効果はあるが、ホスティング層でルーティングを書く必要があり構成が複雑になる。

本連載では「ホスティング設定が直感的」「Astro の `site` 設定がアプリごとに完結する」という理由でサブドメイン分割を採用する。

## ホスティング先

本連載でも Cloudflare Pages を採用する。理由は単一プロジェクト編と同じで、無料枠が広く、サブドメインごとに別 Pages プロジェクトを作って同じリポジトリを接続する、というモノレポ向きの運用がしやすいからである。各サイトの設定を「Root directory は `/`、Build command は `pnpm --filter <app> build`、Build output directory は `apps/<app>/dist`」と書けば済む。

## 次回に向けて

次回は実作業に入る。`mkdir my-project-monorepo` から始めて、`pnpm init` で workspace を初期化し、`pnpm-workspace.yaml` の作成までを行う。各種コマンドとその出力を逐一示しながら進める。
