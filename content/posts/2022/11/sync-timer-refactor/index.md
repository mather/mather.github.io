---
title: "SyncTimerのリファクタリング"
date: 2022-11-15T22:32:53+09:00
categories:
  - SyncTimer開発
tags:
  - 技術
  - SyncTimer
  - Elm
---

[SyncTimerのコードをリファクタリング](https://github.com/mather/sync-timer/pull/45)してみた。

リファクタリングとは、動作を変更させずにコードを整理する作業のことである。
繰り返している無駄な機能をまとめたり、わかりやすく整理することで保守しやすくすることができる。

<!--more-->

## リセット時の初期秒数を Setting にまとめる

作るときの順番としてたまたま `initialTimeSeconds` の値が `Model` に含まれておりそのままにしていたのだが、
文字色や背景色の設定を追加されたことで複数の設定項目をまとめたくなり、 `Setting` という型を作成してまとめた。

内容的に `Setting` がタイマーの表示の設定値のみであるように思っていたのだが、
よくよく考えるとクエリ文字列に保存している値を作るために `initialTimeSeconds` と渡しているので、これもいわゆる設定値である。

```elm
urlFromConfig : String -> BgColor -> Int -> String
urlFromConfig fg bg initialTimeSeconds = ...
```

ということで、この後のことも考えて設定値全般を `Setting` にまとめることにした。


## 設定の初期値をまとめる

これまでクエリ文字列をパースしたときに設定値が指定されていない場合に初期値を設定するように定義していた。
しかし、当初よりも設定情報の種類が増えそうな予感がしたので、
あちこちに散らばるより `Setting` 型の初期値 `defaultSetting` を具体的に一つ定義してあげたほうが適切だと思い、
一つの値として整理した。

```elm
defaultSetting : Setting
defaultSetting =
    { fgColor = "#415462"
    , bgColor = GreenBack
    , initialTimeSeconds = -10
    }
```

## Url.Parser.Query の仕組みを上手く使う

クエリ文字列からパラメータを取り出す処理に [`Url.Parser.Query`](https://package.elm-lang.org/packages/elm/url/latest/Url-Parser-Query) を使っているが、
これまでは `Query.string` などの関数を用いて得られる `Maybe String` などの値をそのまま利用していた。

```elm
Query.string : String -> Parser (Maybe String)
```

そのため、得られる値を定義した型 `InitParams` が `Setting` とほぼ同じ内容ですべて `Maybe` がついた状態となっていた。

```elm
type alias InitParams =
    { fgColor : Maybe String
    , bgColor : Maybe BgColor
    , initialTimeSeconds : Maybe Int
    }
```

しかし、改めて考えてみると `Parser (Maybe String)` に初期値を加えてあらかじめ `Parser String` に変換しておけばよい。
ということで以下の処理を加えた。

```elm
parserWithDefault : a -> Query.Parser (Maybe a) -> Query.Parser a
parserWithDefault default =
    Query.map <| Maybe.withDefault default
```

`Query.map : (a -> b) -> Parser a -> Parser b` を用いることで「存在しなければ初期値を返す」という関数適用を行えば、`Maybe` ではなく確実に値を返すパーサーが出来上がる。

これを最後に [`Query.map3`](https://package.elm-lang.org/packages/elm/url/latest/Url-Parser-Query#map3) を使って `Setting` に変換することができる。

```elm
queryParser : Query.Parser Setting
queryParser =
    Query.map3
        Setting
        (parserWithDefault defaultSetting.bgColor <| Query.enum "bg" dictBgColor)
        (parserWithDefault defaultSetting.fgColor <| Query.string "fg")
        (parserWithDefault defaultSetting.initialTimeSeconds <| Query.int "init")
```

ここで `Setting` そのものは本来は型エイリアスなのだが、3つの値を受け取って `Setting` を返すコンストラクタとしても使えることに留意する。

これにより `InitParams` という中間処理の型が不要になった。

## ネストしたモデルの更新方法

そんな文法があることを知らなかっただけなのだが、ネストしたモデルの更新などに使える方法を見つけたので使ってみた。

これまで、 `update` には `Model` の値をそのまま渡しており、その内部の `Setting` の値を更新したいときは専用の関数を作ってこんな風にしていた。

```elm
{ model | setting = updateBgColor bgColor model.setting }

updateBgColor : BgColor -> Setting -> Setting
updateBgColor bgColor setting =
    { setting | bgColor = bgColor }
```

しかし、関数の引数で受け取るときに `{ setting } as model` という形でネストした値についても変数の束縛ができるのである。

https://faq.elm-community.org/#how-can-i-pattern-match-a-record-and-its-values-at-the-same-time

そんなわけで、これを使えばわざわざ専用の関数を作る必要はなくシンプルに記述できた。

```elm
update msg ({ setting } as model) = ...

{ model | setting = { setting | bgColor = bgColor } }
```

「みんな面倒に思うポイントだしきっと良い解決方法があるはずだ」と思って探してみれば、やっぱりあるもんだ。納得するまでしつこく調べてみよう。

## リファクタリングは大事

その他の細かい修正点もあるが、おおよそ上記のような修正を行った。

リファクタリングは機能を追加することもなく変更を行うため、
ビジネスの現場では「生産性がない」「変更したことでバグを生むリスクがある」という考え方で避けられがちなのだが、
以下の面でメリットが大きいと思う。

- 改めてコードを見ることで理解を深め、もっと良い方法に気づく
- 出来上がっている機能から本質的な問題（今回でいうと初期時間が `Setting` に含まれるべきであること）に還元できる

本来はリファクタリングを行う上で動作するテストを記述し「動作が変わっていないこと」を担保するのが良いが、
SyncTimerの場合はそもそも機能が少ないことやElmの純粋関数型の利点もあり十分信頼できるのでリファクタリングを行った。
