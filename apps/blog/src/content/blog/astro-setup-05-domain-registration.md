---
title: 'Astroで個人サイトを作る #5 ドメイン取得とDNS設定'
description: '独自ドメインを取得し、Cloudflare Pages に紐付けて HTTPS を有効化する。DNS の基礎やリダイレクト戦略も合わせて整理する。'
pubDate: 'Apr 18 2026'
---

> 本記事は単一プロジェクト構成の連載の最終話である。複数サイトを 1 リポジトリで運用したい場合は「Astroモノレポ構成編」を参照のこと。

連載最終回は、デプロイしたサイトを独自ドメインで公開する工程を扱う。`<project>.pages.dev` のような自動付与の URL でも公開はできるが、長期的に運用するなら自分の住所を持っておきたい。

## ドメイン名の構造をおさらいする

ドメイン名は右から左に階層を持つ。`blog.example.com` を例にすると、

- ルートドメイン: `.`（暗黙）
- TLD（トップレベルドメイン）: `com`
- セカンドレベルドメイン: `example`
- サブドメイン: `blog`

という構造になる。TLD には `.com` のような gTLD と、`.jp` のような ccTLD がある。価格、レジストリの方針、信頼感、用途の三軸で選ぶことになる。

個人サイトでよく使われる TLD は次あたり。

- `.com`: 一番無難。ただし良い名前は枯れている。
- `.dev`, `.app`, `.page`: Google が運営。HSTS preload が初期から有効で、全通信が強制 HTTPS になる。
- `.io`: 開発者文化に好まれてきたが値上げ傾向。
- `.me`, `.xyz`: 自由度が高く、安価なものを探しやすい。

## レジストラとレジストリの違い

ドメインを取得するときに登場する用語は紛らわしい。整理すると次のようになる。

- **レジストリ**: その TLD を管理する団体。`.com` なら Verisign、`.jp` なら JPRS。
- **レジストラ**: ユーザーにドメインを販売する事業者。Cloudflare Registrar、Porkbun、お名前.com など。
- **リセラ**: レジストラからさらに販売を委託される業者。

長期運用ではレジストラの安定性が効いてくる。次のような観点で選ぶとよい。

- 更新時の価格が初年度と乖離していないか。
- WHOIS プライバシー保護が無料で付くか。
- 二要素認証、ドメインロック、Transfer Lock が標準装備か。
- DNSSEC やワンクリック移管に対応しているか。

価格面では Cloudflare Registrar が「卸値での販売」を謳っており、無駄なマージンが乗っていない。`.com` であれば年 1,000〜1,500 円程度に収まる。本連載で Cloudflare Pages を使う前提なら、レジストラも Cloudflare に揃えるのが扱いやすい。

## DNS とは何を解決しているか

ドメインを取ったあとに必要なのが DNS 設定である。DNS は「ドメイン名から IP アドレスやサービス名を引く分散データベース」と言える。よく使うレコードタイプは次の通り。

- **A**: ドメイン → IPv4 アドレス
- **AAAA**: ドメイン → IPv6 アドレス
- **CNAME**: ドメイン → 別のドメイン名（エイリアス）
- **MX**: メール配送先
- **TXT**: 任意の文字列。SPF/DKIM/DMARC、各種所有権検証に使う
- **NS**: そのゾーンの権威 DNS サーバ
- **CAA**: 証明書を発行できる CA を制限する

静的サイトのホスティングは「このサブドメインに対しては CNAME を `<project>.pages.dev` に向けてください」のような指示を出してくる。レジストラの DNS で設定するか、別の DNS（Cloudflare DNS、Route 53 など）に NS を切り替えて設定するかを選ぶことになる。

## Cloudflare Pages への紐付け

具体的な接続例として、Cloudflare Pages にドメインを紐付ける流れを示す。

1. Cloudflare ダッシュボードでドメインをサイトとして追加する。レジストラ側で NS を Cloudflare のものに切り替える（Cloudflare Registrar で買った場合はこの手順は不要）。
2. Pages プロジェクトの Custom domains を開き、「Set up a custom domain」をクリック。
3. 公開したいドメイン（例: `example.com` または `blog.example.com`）を入力する。
4. Cloudflare が必要な DNS レコードを自動で作成し、TLS 証明書を発行する。
5. 数分待つと HTTPS で接続できるようになる。

Apex（`example.com` のような頂点ドメイン）も、Cloudflare の CNAME Flattening のおかげで CNAME のように扱える。サブドメインのみのときは通常の CNAME になる。

## HTTPS と Let's Encrypt

主要なホスティングサービスは、独自ドメインに対して自動的に Let's Encrypt の証明書を発行する。証明書は 90 日で期限切れになるため、ホスティング側が自動で再発行を行う。利用者側で意識する必要はほぼないが、CAA レコードを設定している場合は、許可する CA に Let's Encrypt（`letsencrypt.org`）を含めておかないと発行に失敗する点に注意が要る。

`.dev` や `.app` のような Google 系 TLD は HSTS preload に登録されており、HTTPS でしか接続できない。これは強力なセキュリティ保証である一方、誤って HTTP リダイレクトの設定をしていると一切繋がらなくなる、というハマりポイントでもある。

## www あり/なしと Apex の扱い

`example.com` と `www.example.com` のどちらを正式とするかは最初に決めておくべきである。両方繋がる状態を放置すると、SEO 的に重複コンテンツとみなされる可能性があり、解析の数字も二分されてしまう。

判断は宗教論争になりがちだが、現代ではホスティング都合で「Apex は Cloudflare の CNAME Flattening で扱える」「`www` は CDN 都合で技術的にやや扱いやすい」のいずれかに寄せる、というのが現実的な落としどころである。

どちらを正式とするにせよ、もう一方からの 301 リダイレクトを忘れずに設定する。Cloudflare Pages なら「Bulk Redirects」、Vercel なら `vercel.json` の `redirects` で書ける。

## サイトマップと検索エンジンへの登録

公開した段階で、サイトマップを検索エンジンに登録しておくのが定番である。Astro の `@astrojs/sitemap` を入れていれば、ビルド時に `dist/sitemap-index.xml` が自動生成される。ビルドログには次のような表示が出る。

```text
[@astrojs/sitemap] `sitemap-index.xml` created at `dist`
```

- **Google Search Console**: ドメインプロパティを登録し、サイトマップ URL を送信する。所有権検証は TXT レコードで行う。
- **Bing Webmaster Tools**: Google Search Console から設定をインポートできる。
- **IndexNow**: Bing / Yandex などが対応。新規記事の URL を push 通知できる仕組みで、クロール待ち時間を短くできる。

`robots.txt` で `/sitemap-index.xml` の場所を明示しておくと、対応クローラーが自動的に拾ってくれる。

## 連載のまとめ

ここまでで「Node を入れる → Astro でプロジェクトを作る → カスタマイズする → GitHub に push → Cloudflare Pages にデプロイ → 独自ドメインを紐付ける」の流れが一周した。個人サイトを 1 プロジェクトとして立ち上げ、運用に乗せる最小限の知識が揃ったことになる。

次のステップとして、ブログ・技術メモ・アプリ用サイトといった複数サイトを 1 つのリポジトリで運用したくなったら、「Astroモノレポ構成編」へ進むとよい。pnpm workspace を使い、サブドメインごとに別プロジェクトとしてデプロイする構成を扱っている。
