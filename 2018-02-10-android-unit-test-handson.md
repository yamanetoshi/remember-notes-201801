# Android Unit Testing Hands-On

以下なネタでもくもくの方向ですがどうなるか。

- [DroidKaigi 2018 - Android Unit Testing Hands-On](https://github.com/srym/DroidKaigi2018UnitTestHandOn)
- [DroidKaigi2018「はじめてのUnit Test」の資料事前共有とフィードバックのお願い](http://fushiroyama.hatenablog.com/entry/2018/02/02/191938)

<!-- more -->

とりあえず
--------------

課題は Java で、らしい。資料として Kotlin で、というのも README に記載あり。

- [Kotlin でテストを書く](https://speakerdeck.com/numa08/kotlin-detesutowoshu-ku)

最後にある宣伝、な書籍は入手必要なのかどうか。

- [Android アプリ開発のための Kotlin 実践プログラミング](https://www.amazon.co.jp/exec/obidos/ASIN/479805366X/yamanetoshi-22)

とりあえず[リポジトリ](https://github.com/srym/DroidKaigi2018UnitTestHandOn)を clone して開始。branch は現状こうなっています。

```
$ git branch -a
* tasks
  remotes/origin/HEAD -> origin/tasks
  remotes/origin/master
  remotes/origin/tasks
```

master に答え、らしい。ウォーミングアップ、から着手します。

ウォーミングアップ
-------------------

はじめよう、と思ったら AndroidStudio 方面で色々設定が足りていない。自宅でやっときゃ良かったよ、と思いつつ時が過ぎていく。

試験実行する設定に手間どったので以下に控えを。

- Run -> Edit Configuration から Android JUnit を追加
- Use classpath of module にて app 指定
- Class が選択できるはずなので Sandbox 指定

で試験実行できました。試験書くの楽しいですね。

Mockito
------------------

さすがにこのあたりからさくさく進めることが微妙になりはじめていたり。とは言いつつも 6. まではなんとかなりました。このハンズオン課題、良いですね。

そういえば
----------------

_もきーと_ではなくて_もっく伊藤_と読んでいたりして (ぇ