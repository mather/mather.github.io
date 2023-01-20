---
title: Elmアプリケーションとしての規模を小さくする
date: 2022-12-04T17:00:00+09:00
tags:
  - 技術
  - Elm
  - SyncTimer
---

SyncTimerのリファクタリングを行った。今回のテーマは「どこまでをElmで管理すべきか？」ということ。

<!--more-->

## 疑問：リンククリックの挙動は管理されるべきか？

紹介ページを作ったことで2ページ目への遷移を行うようになったが、ここで一つ気になるポイントが生まれた。
単純なリンクのクリックであっても、 `Browser.application` では `Msg` として管理しなければならないという回りくどい部分だ。

https://github.com/mather/sync-timer/blob/bb98a2ad01d76184d9073f9a85a867e09d754a0b/src/Main.elm#L499-L501

これまでのリンクはすべて新規タブで開くようにしていたので該当しなかったが、
 [`Browser.application`](https://package.elm-lang.org/packages/elm/browser/latest/Browser#application) はSPAのような挙動を想定して作られているため、
リクエストされたURLをアプリケーションで処理すべきか、ブラウザ上のページ遷移として扱うべきかをコントロールする必要があるのだ。

そもそもなぜ `Browser.application` を使っていたのかというと、設定変更時にブラウザのURLを書き換えて反映させるためだった。
では、その機能や静的なコンテンツであるヘッダー・フッターを HTML、JS 側に移動させるとElmが管理すべき対象はどのくらいシンプルになるのか？

## アプリケーションの挙動として管理しなくてよい部分

- `Browser.application` と `Browser.document` はページ全体を管理するためHTMLのタイトルなども管理対象だったが必要ない。
- ヘッダー部分、フッター部分はタイマーの挙動とは関係なく固定されている。
  
上記のような見直しの結果、タイマーの表示・コントロール・設定の部分のみを切り出して [`Browser.element`](https://package.elm-lang.org/packages/elm/browser/latest/Browser#element) で実装することにした。
ただし、このままではブラウザのURLを設定に合わせて変更させる機能が動かなくなるので、JSを使って連動させることにした。

## 読み込み時の初期値をElmに取り込む

`Browser.element` ではリクエストされたURLを取得することはできないので、JSで取得してElmアプリケーションに渡す必要がある。
そのため、以下のようなコードを記述してクエリ文字列のパースを行うことにした。

```js
const parseParams = () => {
  const searchParams = (new URL(document.location)).searchParams;
  const parseFg = (s) => {
    if (!s) return null;
    const re = /^#[0-9a-f]{6}$/;
    if (re.test(s)) return s;
    return null;
  }
  const parseInit = (s) => {
    if (s) {
      const n = parseInt(s, 10);
      if (n > 30 || n < -30) return null;
      return n;
    }
    return null;
  }
  return {
    fg: parseFg(searchParams.get("fg")),
    bg: searchParams.get("bg"),
    init: parseInit(searchParams.get("init")),
    h: searchParams.get("h")
  };
};


const app = Elm.Main.init({
  node: document.getElementById("root"),
  flags: parseParams()
});
```

RGB値、数値に関してはバリデーションや型変換を行い、 Elmアプリケーションの `flags` にシンプルなJavaScriptオブジェクトとして渡すだけ。
仮にパースに失敗しても、 `null` を渡すことでデフォルト値を採用するようにしている。

Elm側は `flags` の値を次のように処理している。

```elm
type alias SettingFromQuery =
    { fg : Maybe String
    , bg : Maybe String
    , init : Maybe Int
    , h : Maybe String
    }

parseSettingFromQuery : SettingFromQuery -> Setting
parseSettingFromQuery setting =
    { fgColor = setting.fg |> Maybe.withDefault defaultSetting.fgColor
    , bgColor = setting.bg |> Maybe.andThen decodeBgColor |> Maybe.withDefault defaultSetting.bgColor
    , initialTimeSeconds = setting.init |> Maybe.withDefault defaultSetting.initialTimeSeconds
    , showHour = setting.h |> Maybe.andThen decodeShowHour |> Maybe.withDefault defaultSetting.showHour
    }

initialModel : SettingFromQuery -> ( Model, Cmd Msg )
initialModel setting =
    let
        initSetting =
            parseSettingFromQuery setting
    in
    ( { timeMillis = initSetting.initialTimeSeconds * 1000
      , paused = True
      , current = Nothing
      , setting = parseSettingFromQuery setting
      }
    , Cmd.none
    )
```

`Browser.element` では `flags` は型引数なので、入力値の型を `SettingFromQuery` と定義する。
もしこの型に構造や値が合致しないデータがElmアプリケーションの起動時に渡された場合、Elmアプリケーションはエラーを起こし起動しない。実に潔い。

## 設定変更を Ports で送信する

すでに Google Analytics の際に Ports を使ってElmアプリケーション外部への挙動は実装していたが、今回はURLを書き換える機能を呼び出す Ports を作成する。

Elm側は設定の変更時に `setQueryString`, `urlFromSetting` 関数を呼び出している。(背景色変更の例)

```elm
        SetBgColor bgColor ->
            ( { model | setting = { setting | bgColor = bgColor } }
            , setQueryString <| urlFromSetting { setting | bgColor = bgColor }
            )
```

これらはそれぞれ次のように定義されている。 `port setQueryString : String -> Cmd msg` が Ports として外部に実装されている関数を呼び出すことを宣言していることになる。

```elm
port setQueryString : String -> Cmd msg

urlFromSetting : Setting -> String
urlFromSetting setting =
    UB.toQuery
        [ UB.string "fg" setting.fgColor
        , UB.string "bg" <| encodeBgColor setting.bgColor
        , UB.int "init" setting.initialTimeSeconds
        , UB.string "h" <| encodeShowHour setting.showHour
        ]
```

JS側では受け取ったクエリ文字列をURLにセットしている。

```js
app.ports.setQueryString.subscribe((newQS) => {
  const currentUrl = new URL(document.location);
  const newUrl = currentUrl.origin + currentUrl.pathname + newQS;
  window.history.replaceState(null, "", newUrl);
})
```

これで一応同じ挙動を実装することができた。

## Elmの行数

修正の結果、 `src/Main.elm` の行数は567行から486行に減った。
そんなに減ってないように見えるが、Elmアプリケーションとして管理する範囲がシンプルになったのと、
アプリケーションの挙動に影響しないヘッダー・フッター部分の修正が `Main.elm` で行われないことがわかっているとすごく楽に感じられる。
