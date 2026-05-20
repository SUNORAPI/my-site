---
title: 'Astroで個人サイトを作る #1 環境構築編'
description: 'Node.js と fnm を整え、Astro プロジェクトを始める前段階を作る。シリーズ初回として選定理由と周辺知識も整理する。'
pubDate: 'Apr 14 2026'
series: 'Astroで個人サイトを作る'
seriesOrder: 1
---

> 本連載は「1 リポジトリで 1 つの Astro サイト」を立ち上げる構成を扱う。将来的にブログ・技術メモ・アプリ用サイトなど複数サイトを 1 リポジトリで運用したい場合は、別連載「Astroモノレポ構成編」を参照されたい。シンプルさを優先するなら本連載、最初から複数サイトを見据えるならモノレポ編、というのが選び方の目安になる。

個人サイトや技術ブログを Astro で立ち上げる連載の初回として、まずは開発環境を整える工程をまとめる。ツールの選定理由や周辺知識にも踏み込み、なぜこの構成にするのかを丁寧に確認していく。

## なぜ Astro なのか

Astro は「コンテンツ志向のサイト」を作るために最適化されたフレームワークで、ブログやドキュメント、ポートフォリオのようにマークアップ中心のサイトに向いている。React や Vue、Svelte などのコンポーネントを混在させて使える一方、デフォルトではクライアントへ JavaScript を送らない「ゼロ JS」がベースであり、Lighthouse スコアを稼ぎやすいのが特徴である。

似た用途のツールには Next.js、Nuxt、Hugo、Eleventy などがあるが、Astro は次の点で選ばれることが増えている。

- マークダウン / MDX のサポートが標準で組み込まれており、コンテンツコレクションという仕組みで型付きの記事メタデータを扱える。
- アイランドアーキテクチャによって、必要な部分だけインタラクティブにできる。
- ビルド出力は静的 HTML が基本で、CDN との相性が良くデプロイ先の選択肢が広い。

「速く軽い静的サイトを型安全に作りたい」というニーズに対しては、現状もっとも素直に当てはまるのが Astro と言える。

## Node.js のバージョンマネージャを選ぶ

Astro は Node.js 上で動く。Node のバージョン切り替えを楽にするバージョンマネージャはいくつか選択肢があり、次のいずれかから選ぶことになる。

- **fnm**: Rust 製で起動が速い。Windows / macOS / Linux いずれでも動く。
- **nvm**: 老舗。シェル起動が遅くなる傾向があり、Windows 非対応。
- **Volta**: `package.json` にバージョンをピン留めできる。チーム開発で揺れを抑えやすい。
- **Docker**: コンテナで隔離。Node を入れるためだけにはオーバースペック。

シンプルに「Node を切り替えたいだけ」であれば fnm が最も摩擦が少ない。本連載では fnm を採用する。

## Node のインストール

macOS で fnm を入れる場合は Homebrew 経由が早い。

```sh
brew install fnm
```

シェル設定に 1 行追加する。zsh ならこうなる。

```sh
echo 'eval "$(fnm env --use-on-cd)"' >> ~/.zshrc
source ~/.zshrc
```

Linux なら次のとおり。

```sh
curl -fsSL https://fnm.vercel.app/install | bash
```

Windows（PowerShell）は winget が手軽である。

```powershell
winget install Schniz.fnm
```

導入が済んだら Node をインストールする。

```sh
fnm install --lts
fnm use lts-latest
```

最後にバージョン確認。

```sh
$ node -v
v22.12.0
$ npm -v
10.9.0
$ fnm --version
fnm 1.39.0
```

Node 22 系の LTS が入っていれば、Astro 5 系の動作要件を満たす。

## エディタとフォーマッタの周辺整備

Astro の `.astro` ファイルは独自のシンタックスを持つため、エディタの拡張を入れておくと作業効率が大きく変わる。VS Code を使うのであれば、公式の Astro 拡張、ESLint、Prettier の三点セットを最初に有効化しておきたい。Prettier は `prettier-plugin-astro` を追加すると `.astro` ファイルを整形できるようになる。

TypeScript のチェックは `astro check` コマンドで行える。CI で型エラーを検出する仕組みも、最初から組み込んでおくと後で痛い目にあわなくて済む。

## Git とリポジトリの準備

開発を始める前に、リポジトリ運用の方針も決めておくと迷いが減る。

- **ブランチ戦略**: 個人サイトなら `main` 一本でも十分。複数人なら `main` + feature ブランチが無難。
- **コミット規約**: Conventional Commits（`feat:`, `fix:`, `docs:` など）を導入しておくと、後から自動リリースツールに接続しやすい。
- **`.gitignore`**: Astro 公式テンプレートには `dist/`, `node_modules/`, `.astro/` などが含まれている。OS 由来の `.DS_Store` や IDE 設定ディレクトリを足しておくとなお安心。

## なぜ Cloudflare Pages を併用するのか

本連載はホスティング先として Cloudflare Pages を想定する。理由は次の三点である。

- 帯域無制限・無料枠が広く、個人サイトであれば実質コストはドメイン代だけで済む。
- Astro の静的出力との相性が良く、GitHub 連携でビルドが自動化される。
- 後からエッジ関数や R2 ストレージなどに繋ぎ足しやすい。

同等の選択肢には Vercel、Netlify、GitHub Pages があるが、連載中はそれらの違いにも触れていく。本記事の時点では「Cloudflare Pages に出すつもりで進める」とだけ覚えておけばよい。

## ここまでの到達点

ここまでで、Astro プロジェクトを始めるための土台が整った。次の記事では `npm create astro@latest` を実行してプロジェクトを生成し、`localhost:4321` で動かすところまで進める。
