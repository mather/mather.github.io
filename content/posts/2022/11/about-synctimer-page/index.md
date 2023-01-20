---
title: SyncTimer の紹介ページを作った
date: 2022-11-27T18:40:11+09:00
categories:
  - SyncTimer開発
tags:
  - 技術
  - SyncTimer
  - JavaScript
---

タイトルそのままなのだが、これまでモーダルとして表示していた使い方動画なども紹介ページへ移動してみた。
これはもともと存在していたのだが、Github Pages として公開しておりアプリケーションと同じドメインではなかったので、同じドメインに `/about` として表示するようにした。

https://sync-timer.netlify.app/about/

モーダルのほうがわかりやすかっただろうか？という疑問もあるが、一旦公開することにした。

<!--more-->

## 紹介ページで使用している技術

[Bulma](https://bulma.io/) というCSSフレームワークを使用している。

[タブ表示](https://bulma.io/documentation/components/tabs/) の見た目についてはこのフレームワークでも対応しているのだが、
実際に動かすためにはJavaScriptコードを書かないといけなくなる。

検索したらすぐ出てくると思うのだが、実例としてここにメモしておこう。

HTML上はタブで切り替えるコンテンツに `id` を振ってあり、一つだけ表示状態にしてある。

```html
<div class="tab-content" id="about-content">
  ...
</div>
<div id="caution-content" class="tab-content" style="display: none">
  ...
</div>
<div class="tab-content" id="inquiry-content" style="display: none">
  ...
</div>
```

タブの方はBulmaのスタイルを適用し、ボタンのように機能するようにする。

```html
<div class="container tabs is-centered is-large is-boxed">
  <ul>
    <li id="about-tab" class="is-active">
      <a onclick="activateTab('about')">概要</a>
    </li>
    <li id="caution-tab">
      <a onclick="activateTab('caution')">利用時の注意</a>
    </li>
    <li id="inquiry-tab">
      <a onclick="activateTab('inquiry')">お問い合わせ</a>
    </li>
  </ul>
</div>
```

タブがクリックされると、 `activateTab` 関数を呼び出してタブの表示を切り替える。
指定されたクラスにマッチするHTMLエレメントを探して、一旦非表示に切り替えて、指定されたものだけ表示状態に切り替えているシンプルな実装だ。

```js
function activateTab(to) {
  const tabs = document.querySelectorAll(".tabs li");
  const tabContents = document.getElementsByClassName("tab-content");
  for (const tab of tabs) {
    tab.className = "";
    if (tab.id == to + "-tab") tab.className = "is-active";
  }
  for (const content of tabContents) {
    content.style.display = "none";
    if (content.id == to + "-content") content.style.display = "block";
  }
  gtag("event", "activate_tab", { event_label: to });
}
```

ただ、難点が一つ。 `display: none;` を適用して切り替えるまで、 `<iframe>` で読み込んでいる要素がロードされないのだ。

ユーザーとしては必要ない読み込みが減って良いようにも思えるが、問い合わせフォームをタブで表示すると一瞬遅れて表示される結果になってしまう。

微妙にユーザー体験が良くない気がするが、そもそも問い合わせフォームが使われたことがないので許容範囲としている。