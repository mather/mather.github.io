---
title: "開発したもの"
date: 2026-05-03T14:32:28+09:00
---


## SyncTimer

https://sync-timer.netlify.app/

[リポジトリ](https://github.com/mather/sync-timer)

YouTubeなどの同時配信用のタイマーとして開発したもの。無料で提供中。

マイナスからスタートできるのでカウントダウンからそのまま動画再生時間になる。

開発履歴は[ブログ](/tags/synctimer/)にて公開中。

技術スタック：

- [Elm](https://elm-lang.org/)
- Netlify (Hosting + CI)

## LeacTion!

https://leaction.web.app/

[リポジトリ](https://github.com/tegehoge/leaction)

オフライン・オンラインでの勉強会で登壇者へ質問や同意・応援などを送れるサービス。無料で提供中。

LT + Reaction で LeacTion! と命名した。

[Webナイト宮崎](https://tegehoge.connpass.com/)などのイベントの際に使用していた。

特徴：

- イベント管理者だけユーザー登録すれば参加者は自由にログイン不要で投稿できる
- 他の人の投稿に「いいね」を送れる
- リアルタイムに投稿が反映される
- 一つのイベントに複数のトークが設定できるので、話題がゴチャゴチャにならない。

技術スタック：

- [Solid.js](https://www.solidjs.com/)
  - TypeScript
- GitHub Actions
- Firebase
  - Hosting
  - Auth
  - Firestore
