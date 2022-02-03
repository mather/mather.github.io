---
title: "ポートフォリオサイトを Hugo で Github Pages + Github Actions で構築する話"
date: 2019-12-07T23:57:04+09:00
toc: false
images:
tags:
  - 技術
  - hugo
  - advent calendar 2019
---

[宮崎 IT 関連勉強会 Advent Calendar 2019](https://qiita.com/advent-calendar/2019/miyazaki) 8 日目の記事です。

皆さん、Github 活用してますか？

Github には Github Pages という機能があり、静的サイトのホスティングを行うことができます。
特に、 `[アカウント名].github.io` という名前のリポジトリの場合はドメイン直下のページが作成できます。

参考: [GitHub Pages サイトの種類](https://help.github.com/ja/github/working-with-github-pages/about-github-pages#github-pages-%E3%82%B5%E3%82%A4%E3%83%88%E3%81%AE%E7%A8%AE%E9%A1%9E)

それに加えて、[Github Actions が一般に公開されました](https://github.blog/changelog/2019-11-11-github-actions-is-generally-available/)。
これは Push などのイベントをトリガーとしてビルド、テスト、デプロイなどが自動的に実行できる仕組みです。いわゆる CI(継続的インテグレーション)や CD(継続的デプロイ)として機能させることができますし、他にもプルリクのレビュー補助や通知などの機能にも利用できるでしょう。

これらを組み合わせると、以下のようなワークフローが可能になります。

- 静的サイトジェネレーター(Hugo, Jekyll, etc...) でサイトを作る
- ローカルでレビューし、問題なければ指定のブランチにプッシュ
- Github Actions でプッシュを検知し、静的サイトをビルド
- `[アカウント名].github.io` の場合、ビルド完了した静的サイトを `master` ブランチにデプロイ
  - それ以外のリポジトリの場合は `gh-pages` ブランチがデプロイ先となります

今回はこれを [Hugo](https://gohugo.io/) でやってみよう、という話です。

## ポートフォリオを作る

Hugo をインストールします。Mac なので Homebrew でインストールできます。

```
$ brew install hugo
```

今回は空っぽの Jekyll サイト(5 年ほど前に作って放置してた)を削除して作り直しますので、既存のファイルを削除したあとは以下のように Hugo の初期化を行います。

```
$ hugo new site . --force
```

Hugo のテーマはシンプルな [hermit](https://themes.gohugo.io/hermit/) にしてみました。
個人的にはこのくらい落ち着いた色合いが好みです。

インストール手順はテーマのページに記載されているドキュメントのとおりですので割愛します。

設定ファイルである `config.toml` は[テーマのデモページで適用されているもの](https://github.com/Track3/hermit/blob/master/exampleSite/config.toml)を参考に自分のページに合わせて編集します。

トップページのメニューはこれから作るパスを考えながら設定します。

```toml
[menu]

  [[menu.main]]
    name = "Posts"
    url = "posts/"
    weight = 10

  [[menu.main]]
    name = "About"
    url = "pages/about/"
    weight = 20
```

以下のコマンドで Hugo のホットリロードサーバーを起動します。

```
$ hugo server -D
```

`-D` は `draft: true` となっているドラフト記事（本番ではまだ表示しない）もビルドして表示してくれるフラグです。

### 自己紹介を書く

`pages/about/` に自己紹介を掲載するので、以下のコマンドでページを作ります。

```
$ hugo new pages/about.md
```

このコマンドで `content/pages/about.md` が生成されます。
生成されたファイルの内容は `archetypes/default.md` をテンプレートとして生成したものですが、この `archetypes/` フォルダには固定ページ向け、ブログ記事向けなどに切り分けたテンプレートも設置できます。

参考: [Archetypes | Hugo](https://gohugo.io/content-management/archetypes/)

生成された Markdown ファイルに内容を記載して保存すると自動的にビルドが行われ、ブラウザで表示している場合は自動的にリロードされます。

表示された内容を確認し、内容に問題がなければ `draft: true` を削除し `git commit` しましょう。
今回は `hugo` ブランチを作ってコミットします。

```
$ git checkout -b hugo
$ git commit -v
```

## Github Actions の準備

リポジトリメニューの Actions を選択すると Github Actions のワークフロー (workflow) を作成できます。

ワークフローを作るときはすでに用意されているワークフローを参考に生成することもできますが、今回は自分で作るので "Set up a workflow yourself" ボタンを押します。

すると YAML ファイルを作成するエディタとマーケットプレイス(Marketplace)が表示されます。
エディタではワークフロー定義の文法エラーなどを指摘してくれます。
また、Marketplace にはすでに Actions で利用可能なアクションが検索可能な状態になっていますので、目当てのものを探して導入することでワークフローを手軽に構成できます。

今回はエディタ開始時にすでに指定されている `actions/checkout@v1` 以外に、以下の action を利用します。

- `peaceiris/actions-hugo@v2.3.0` : Hugo をインストールする
- `peaceiris/actions-gh-pages@v2.5.1` : Github Pages をデプロイする

これらを使って構成した Workflow は次のようになります。

```yml
name: Deploy Github Pages

on:
  push:
    branches:
      - hugo

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Source
        uses: actions/checkout@v1
      - name: Clone submodule
        run: git submodule update --init --recursive
      - name: Hugo setup
        uses: peaceiris/actions-hugo@v2.3.0
        with:
          # The Hugo version to download (if necessary) and use. Example: 0.58.2
          hugo-version: 0.60.1
          # Download (if necessary) and use Hugo extended version. Example: true
          extended: true
      - name: Build Hugo
        run: hugo -v
      - name: GitHub Pages action
        uses: peaceiris/actions-gh-pages@v2.5.1
        env:
          ACTIONS_DEPLOY_KEY: ${{ secrets.ACTIONS_DEPLOY_KEY }}
          PUBLISH_BRANCH: master
          PUBLISH_DIR: ./public
```

内容は各ステップの `name` にあるとおりです。

- ソースコードをチェックアウト
- submodule をチェックアウト ( `themes/hermit` を submodule で導入したため )
- Hugo をインストール
- Hugo をビルド
- Github Pages へデプロイ

`uses` ではなく `run` になっているステップでは、実際にこのコマンドをシェルで実行するだけです。

作成が完了したら "Start Commit" を押してコミットを作成するのですが、直接 `master` ブランチにコミットするのではなく、 `hugo` ブランチに作成したいのでプルリクエストを作成する方を選びます。

### デプロイキー (Deploy Key) の登録

上記ワークフロー定義の `ACTIONS_DEPLOY_KEY: ${{ secrets.ACTIONS_DEPLOY_KEY }}` となっている部分でデプロイキーの指定が必要なのですが、こちらはローカルで SSH 鍵ペアを生成する必要があります。

以下はパスフレーズなしの鍵ペアを作成するコマンドの例です。

```
$ ssh-keygen -t rsa -b 4096 -f /path/to/key -C "[Githubアカウントのメールアドレス]" -N ""
```

生成した鍵ペアをリポジトリのメニューから以下のように登録します。

- `Setting` -> `Deploy Key` -> 公開鍵 (`.pub` の方) を追加
- `Setting` -> `Secrets` -> `ACTIONS_DEPLOY_KEY` というキー名で秘密鍵を登録

これで準備完了です。 `hugo` ブランチが更新されたら、ワークフローが起動します。

## まとめ

やや操作は多くなりましたが、静的サイトジェネレーターを使って Github Pages を自動デプロイする方法が Github だけで完結するのはとても便利だと思います。

ぜひ皆さんもワークフローを作ってみてください。
