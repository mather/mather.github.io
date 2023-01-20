---
title: "SyncTimer に Google Analytics を導入した"
date: 2022-04-30T19:00:00+09:00
categories:
  - SyncTimer開発
tags:
  - 技術
  - SyncTimer
---

たぶん使ってる人はいるんだろうなー、と思いつつも利用報告をしてくれるユーザーはほぼいない。
それは当然で、報告するメリットもないし、不具合があったり使い勝手が悪いなーと思っても似たような手段はいくらでもあるので別のツールに切り替えるだけになる。
だったら自分で調べるしか無いよね、ということで Google Analytics を導入して実際のアクセスを調査するしか無いのである。

今回は Elm で書いたアプリケーション内のイベントをGA4に送信する実装のお話。
<!--more-->

## イベントを送信する

素直にタグを記述するだけでページへのアクセスなどは取得できるのだが、そもそもSyncTimerは1ページしか無く、
「アクセスがあった」「スクロールした」くらいの情報やOS,ブラウザの種類などが取れる程度なので、
実際にタイマーを動かしたかを知りたい場合はイベントを送信する必要がある。

[GA4 - イベントを設定する](https://developers.google.com/analytics/devguides/collection/ga4/events?client_type=gtag)

Elm側で実行されたイベントを送信するには、Elmコードの外にある `gtag()` 関数を呼び出す必要があるので、ここで Ports という機能を使うことになる。

[ポート(Ports) - Elm](https://guide.elm-lang.jp/interop/ports.html)

簡単に言ってしまえば外部とやり取りする方法で、「外部に文字列(String)を送る」「外部からStringを受け取る」しかない。
今回の場合はイベントを文字列で送るのだが、イベントの種類などの構造を持ったデータを送信したいので JSON 形式に変換する関数を作る。

```elm
import Json.Encode as E

encodeAnalyticsEvent : String -> String -> String -> Maybe Int -> String
encodeAnalyticsEvent category action label value =
    E.encode 0 <|
        E.object <|
            [ ( "category", E.string category )
            , ( "action", E.string action )
            , ( "label", E.string label )
            ]
                ++ (Maybe.map (\v -> [ ( "value", E.int v ) ]) value |> Maybe.withDefault [])
```

で、この関数で生成される文字列を port で外部に送信する

```elm
port sendAnalyticsEvent : String -> Cmd msg

timerStartEvent : Int -> Cmd msg
timerStartEvent currentTime =
    sendAnalyticsEvent <| encodeAnalyticsEvent "sync_timer" "sync_timer_start" (formatTimeForAnalytics currentTime) Nothing
```

外部のJSで受け取る方は JSON 文字列をパースして `gtag()` 関数でイベントを送信する。

```js
app.ports.sendAnalyticsEvent.subscribe((event) => {
  const { category, action, label, value } = JSON.parse(event);
  console.debug({ category, action, label, value });
  if (gtag) {
    const data = Object.assign({
      "event_category": category,
      "event_label": label
    }, value ? { value } : {});
    gtag("event", action, data);
  }
});
```

という仕組みで送信されている。

おかげで利用状況がある程度わかるようになってきた。