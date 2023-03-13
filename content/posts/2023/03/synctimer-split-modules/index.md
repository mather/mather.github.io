---
title: "SyncTimerのコードをいくつかのモジュールに分解した"
date: 2023-03-13T17:28:02+09:00
categories:
  - SyncTimer開発
tags:
  - 技術
  - Elm
---

これまで `Main.elm` だけで作ってきた Elm のコードが増えて見通しが悪くなってきたので、いくつかのモジュールに分解した。
今回はリファクタリングなので機能の変更はなし。

<!--more-->

## モジュールへの分解

分解前のコードが691行になっていた。ここまで増えるとスクロールも疲れるし、どこに何があるかわからなくなってしまう。

そこで、大きな構成要素を整理しモジュールに分けてみることにした。

- `Main.elm`
  -  `main` 関数、および起動時のパラメータ管理。
- `Model.elm` 
  -  `Model` 型を中心としたデータの定義とデータの変換に関する関数群。
  -  `Cmd Msg` , `Html Msg` を含まない純粋なもの。
- `Msg.elm`
  - `Msg` 型、 `update` 関数の定義。
  - ユーザーインタラクションに対するふるまい。
- `View.elm`
  - `view` 関数の定義。
  - 画面表示に関するもの。
- `Analytics.elm`
  - Google Analytics 向けのイベント送信の定義。

どうやって分解するかは我流だが、それぞれのファイルがどのライブラリやモジュールに依存するか見るとだいぶスッキリしたように見える。

### import で見る依存関係

それぞれのファイルから `import` を抜き出してみる。

`Main.elm`

```elm
import Browser
import Browser.Events
import Model exposing (Model, Setting, decodeBgColor, decodeBoolean, decodeFgFont, defaultSetting)
import Msg exposing (Msg(..), update)
import View exposing (view)
```

`main` 関数はすべての起点なので `Model`, `Msg`, `View` のモジュールを使っていることは当然なのだが、
それ以外が `main` 関数を作るための `Browser` と、 `subscriptions` に使う `Browser.Events` だけになっていてかなりシンプルになった。

`Model.elm`

```elm
import Dict
import Time
```

たったこれだけ。他のモジュールへの依存もなく、データを中心とした純粋なモジュールになった。

`Msg.elm`

```elm
import Analytics exposing (timerFastForwardEvent, timerPauseEvent, timerResetEvent, timerRewindEvent, timerStartEvent)
import Dict
import Model exposing (BgColor, FgFont, Model, Setting, defaultSetting, encodeBgColor, encodeBoolean, encodeFgFont)
import Time
import Url.Builder as UB
```

設定の更新時にURLを同期させる目的で `Url.Builder` を使用している以外は、 `View` にも依存しておらず `Model` を中心に動作していることがはっきりしている。

`View.elm`

```elm
import Html exposing (Attribute, Html, a, button, details, div, i, input, label, option, select, span, summary, text)
import Html.Attributes as A exposing (attribute, checked, class, for, id, selected, step, style, type_, value)
import Html.Events exposing (onClick, onInput)
import Model exposing (BgColor(..), FgFont(..), Model, Setting, decodeBgColor, decodeFgFont, encodeBgColor, encodeFgFont)
import Msg exposing (Msg(..))
```

`Model`, `Msg` に依存しているのは言うまでもないが、それ以外は完全に `Html` 関連のインポートだけになっている。
逆に言えば、 `Html` 関連のインポートは `View.elm` だけしか存在しない。

`Analytics.elm`

```elm
port module Analytics exposing (..)

import Json.Encode as E
import Model exposing (Setting, encodeBgColor, encodeFgFont)
```

Analytics に送信するための `port` の定義もここに配置したので最初の宣言が `port module` となっている。
また、 `Json` 関連のモジュールもここでしか使わないことが明確になっている。

### 変更前の import を見てみる

変更前はどうだったかというと、当たり前だが上記を全部放り込んだ形になっている。

```elm
import Browser
import Dict
import Html exposing (Attribute, Html, a, button, details, div, i, input, label, option, select, span, summary, text)
import Html.Attributes as A exposing (attribute, checked, class, for, id, selected, step, style, type_, value)
import Html.Events exposing (onClick, onInput)
import Json.Encode as E
import Time
import Url.Builder as UB
```

どこかで使ってるんだろうな、ということは理解できるものの、やはり見通しが悪くなるのは否めない。
（フォーマッタがよしなに整理してくれて使ってないモジュールもエディタがグレーアウトしてくれるのでなんとかなっているが）

## スコープは小さく、依存も小さく

アプリケーションコードが持っている構造を見えるように整理することは読み手を手助けしてくれるし、一つのモジュールについて集中して考えるときに余計なコードが目につかなくなる。

また、モジュールに分解されたことで変更履歴の差分をファイル一覧で見るだけでもどこに変更が加わったか予想しやすくなったりする。

この規模のコードで一人で開発している分にはメリットはあまりないと思うかもしれないが、逆に大規模になったりチームで開発するときはこのようなリファクタリングを行うタイミングすら少なくなり、
「いつかコードを整理したい」と思っていても練習すらままならない状態でいきなり取り掛かることも難しいので実現できないリファクタリングをいつまでも夢見ることになる。

コードの構造に注目して分解するなどのリファクタリングの実践練習は自己責任で変更できる小さいプロダクトで試しておくのがいいかもしれない。
