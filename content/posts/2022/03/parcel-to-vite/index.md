---
title: "Parcel to Vite"
date: 2022-03-15T21:00:00+09:00
draft: true
categories:
  - SyncTimer開発
tags:
  - 技術
  - SyncTimer
---

SyncTimerのビルドツールはこれまで Parcel2 を使ってたが、開発時のサーバーの動作がおかしく開発するのに不便なことが何度かあった。

そこで、ビルドツールを Vite に変更することにした。
<!--more-->

{{< tweet user="mather314" id="1503603626568466433" >}}

## Parcel

[Parcel](https://parceljs.org/) はゼロコンフィギュレーション（基本的に設定ファイルが不要）を売りにしていて、Elmにも標準対応している強みがあった。

とはいえ実際のところは、静的な画像ファイルなどの転送のために `parcel-reporter-static-files-copy` プラグインを入れて `.parcelrc` という設定ファイルで有効にする必要があったりして、
ちょっと癖はあるなーと思っていた。

そんな中、あるとき起動しているはずの開発サーバーにアクセスできない謎の事象が発生したのでそのうち修正されないかなーと思いながら
質問だけ送っていた。

https://github.com/parcel-bundler/parcel/discussions/7757

しかし、一ヶ月近く経過しても反応は無いので、別のビルドツールを試してみることになったのだった。

## Vite

ビルドがすごい早いと謳っている [Vite](https://ja.vitejs.dev/) を別のプロジェクトで使ってたので、こちらに [Elmプラグイン](https://github.com/hmsk/vite-plugin-elm) を導入して構築。

プラグインを入れるために設定は追加したのだが、これはゼロコンフィギュレーションと言ってもいいレベルなので Parcel とほぼ変わらず。

```ts
import { defineConfig } from 'vite'
import elmPlugin from 'vite-plugin-elm'

export default defineConfig({
  plugins: [elmPlugin()]
})
```

その他、ElmだけではなくCSSフレームワークなども `import` で記載することで一つのCSSファイルとしてバンドルしてくれるようになったのでちょっと便利になった。

ありがたいことです。

## 移行完了

機能はいじってないので利用者には何も関係ないけど、ちょっと開発がやりやすくなった。
