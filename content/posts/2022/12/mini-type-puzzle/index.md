---
title: "Mini Type Puzzle"
date: 2022-12-07T17:39:53+09:00
summary: Elm アドベントカレンダー 2022 の記事。 型がカチッとはまって意図通りの関数が作れたときは気持ちいいよね。
tags:
  - 技術
  - Elm
---

飛び入りで [Elm アドベントカレンダー 2022](https://qiita.com/advent-calendar/2022/elm) の12/7の記事を書いてみようと思う。

とはいえ何も準備していないので、最近の気持ちよかったことを書く。

## 関数がピッタリハマったときの気持ちよさ

[趣味で作っているタイマー](https://github.com/mather/sync-timer)のコードをぼちぼちいじってたときに、ラジオボタンをセレクトボックスに変更しようと思って書き換えていた。

これまでのラジオボタンなら、各 `input` の属性に `onClick <| SetBgColor GreenBack` などのように個別に記述すればよかったのだが、 `select` になると `onInput : String -> Msg` を使う必要が出てくる。

でまぁ文字列を変換するコードをちまちま書けばいいじゃないか、という話なのだが、今回はこれまでに書いていたいくつかのパーツがきれいにハマって書くことができたのでちょっと嬉しかったのだ。

あらかじめ言っておくと、タイトルにいうほどのパズルではないと思うので期待はしないでいただきたい。

### 文字列を代数的データ型に

設定情報をクエリ文字列に変換し、クエリ文字列から設定を復元する、という仕様のため、文字列から代数的データ型の変換を行っている。
このとき便利なのが `Dict` で、次のように対応関係を定義していた。

```elm
type BgColor
    = Transparent
    | GreenBack
    | BlueBack

dictBgColor : Dict.Dict String BgColor
dictBgColor =
    let
        pairwise bgColor =
            ( encodeBgColor bgColor, bgColor )
    in
    Dict.fromList <| List.map pairwise [ GreenBack, BlueBack, Transparent ]
```

これは以前 [`Url.Parser.Query.enum`](https://package.elm-lang.org/packages/elm/url/latest/Url-Parser-Query#enum) を使っていたときの名残だ。

```elm
enum : String -> Dict String a -> Parser (Maybe a)
```

そもそも、 [`Dict.get`](https://package.elm-lang.org/packages/elm/core/latest/Dict#get) が似たようなことをしている。

```elm
get : comparable -> Dict comparable v -> Maybe v
```

しかし、この関数の都合の悪いのは引数の順番である。 `Dict comparable v` の部分を固定して使いたいときに不便だ。
そこで、関数型のアイデアとしては `flip : (a -> b -> c) -> b -> a -> c` を使いたくなるのだが、 [Elmではすでに削除されている](https://github.com/elm/compiler/blob/master/docs/upgrade-instructions/0.19.0.md#functions-removed)ので簡単に実装する。

```elm
flip : (a -> b -> c) -> b -> a -> c
flip f a b =
    f b a
```

さて、これによって `String -> Maybe BgColor` という対応関係が作れる

```elm
flip Dict.get dictBgColor -- String -> Maybe BgColor
```


### `Msg` に変換する

`BgColor` の変更を伝える `SetBgColor` が定義されていて、これは `SetBgColor : BgColor -> Msg` とみなせるので、これを `Maybe BgColor` に適用したい。
つまりは `Maybe.map` を使えば良い。

```elm
Maybe.map SetBgColor -- Maybe BgColor -> Maybe Msg
```

### `Maybe Msg` のままでは渡せない

`onInput : (String -> Msg) -> Html.Attribute Msg` のシグネチャには `Msg` が要求されるので最後は `Maybe` ではだめだ。

実際にはそんなケースは発生しないことはわかっているのだが、入力が `String` である以上はイレギュラーなケースもカバーしなければならない。

そこで、 `NoOp` という値が役に立つ。もし `BgColor` に該当しないイレギュラーな値が入力されたときは最終的に「何もしない」が送信される。

```elm
Maybe.withDefault NoOp -- Maybe Msg -> Msg
```


### 関数合成

3つの関数の入力と出力がうまい具合に噛み合った。あとはこれを合成するだけだ。

```elm
selectBgColor : String -> Msg
selectBgColor =
    flip Dict.get dictBgColor >> Maybe.map SetBgColor >> Maybe.withDefault NoOp
```

カッコもなくスッキリと合成できた。
一つ一つのシンプルな仕組みがうまく合致して整理できたときはやっぱり嬉しいな。
