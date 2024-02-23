# Scala3移行の話
- ぽんこつ
  - @ponkotuy@social.mikutter.hachune.net
  - @ponkotuy(Twitter)
  - @ponkotuy.bsky.social


## 祝 Playアップデート
- 待ちに待ったPlayの新バージョンがリリース
- 2.9と3.0があります
- Scala3を使いたいだけならどちらでもよい


## 社内プロジェクト
- ベースは退社した人が作ったやつ
- よくあるPlayのAPIサーバ
- Scala 2.13 / Play 2.8
- Libアップデートは放置気味
- テストは半分ぐらい網羅

これをScala3が使えるようにする


## Play2.9アプデ
- 全く難しくない(ぶっちゃけ覚えてない)
- Scalaのマイナーアップデートのwarningぐらい
  - Scala3で引っ掛かる部分なのでむしろ便利


## Scala3移行
- 2022年に書かれたスライドの通り大体やった

https://xuwei-k.github.io/slides/alp-scala-3/


おわり


## コンパイルを通す
動かなくていいのでコード書き換えたりしてとにかくコンパイルを通す

(当初予定だと1PRで終わらす予定だったけど終わらないことが発覚してくる)


## 発覚したブロッカー
1. specs2-mock(mockito-scala)
1. circe-generics-extras
1. play-redis
1. mailgun4s

やばそうな順


## specs2-mock
- テストフレームワークspecs2にmockを導入するLib
- mockito-scalaに依存
  - マクロ非互換の問題できつい
  - JDK内部実装依存まであるらしい
  - 待っても対応されなそう
- Scala業界を恐怖のどん底に落とした(かもしれない)


## specs2-mock依存削除
- ベースのMockitoを直で使うように置き換え


#### before
```scala
val hoge = mock[Hoge]
hoge.func returns fuga
there was one(hoge.func)
```

#### after
```scala
val hoge = mock(classOf[Hoge])
when(hoge.func).thenReturn(fuga)
verify(hoge, times(1)).func
true must beTrue
```


## PR(remove specs2 mock)
Files changed(86) +1,346 -1,466

つらい


## circe-generic-extras
- ScalaのJSONライブラリcirce
- circeのdecode/encodeに便利な機能を追加する
- 内部実装はほぼマクロ
- package privateのclass触るためにcirceになってるだけで別プロジェクト


## 社内のcirce-generic-extras
decoderがデフォルト値を使うようになる

```scala
case class Person(name: String = "default")
object Person {
  implicit val c = Configuration.default.withDefaults
  implicit val d = deriveConfiguredDecoder
}
```

name要素無しでパースできるようになる


**POSTのbodyパーサにcirce使ってるって…こと…？**

Play Forms使え案件


## circe-generic-extras依存削除
- decoderを使ってないのは純粋に削除
- デフォルト値が無いのは置き換え
- テストを追加してdefault値動作を保証
- Formsを使うべきだが一旦class増やして逃げる

```scala
case class PersonRequest(name: Option[String]) {
  def build = Person(name.getOrElse("default"))
}
```


## PR(Remove circe-generic-extras)
Files changed(92) +1,142 -600

前の動作を保証するためにテスト追加が多い。つらい


## mailgun4s
- MailgunはメールのSaaS
- MailgunをScalaで使いやすくするLib
- 最新版はScala3に対応している


## アプデすればよくない？
- groupが変更 "org.matthicks" -> "outr"
- APIが大幅に変更
  - cats-effect依存が追加
  - 返り値がFutureからIOに

モナモナしたコードわからんしメンテできん


## Mailgun再実装
- 単純なHTTPのAPIなのでsttpで再実装
- 再実装終わったあたりで別のところでMailgunのsttp実装を発見
- なんで2つもMailgun実装があるんだこのプロジェクト

→最終的に既存実装に寄せて削除


## PR(Remove mailgun4s)
Files changed(19) +33 -172


## 長期メンテ可能なScala
- Scala 2.9|2.10|2.11|2.12|2.13|3
- 既存コードが壊れるアプデはほぼない
  - マクロは大体壊れる
  - Collectionは一度大改修あった
- バイナリ互換はほぼ無い
  - ビルド環境のアップデートが必要
- 常にネックはScala Lib
  - Java Libを使う
  - 長期的にメンテされそうなLibを使う


## 長期メンテされるLib
### マクロを使ってない
### JDKに依存していない
### 黒魔術使ってない
黒魔術を駆使して利便性を上げたLibが爆死するのを沢山みた

ちょっとした利便性のためにマクロ使う、使ったLibを使うのは避けたい


## ビルド環境事情
- Scalaではsbtがデファクト
  - Scala固有事情がありsbt以外考えられない
- アップデートは頻繁
  - 過去には破壊的変更も多数
  - ここ数年はほぼない
- 文法が独特(とっつきにくい)
- 機能的には十分かつ豊富


## まとめ
- scala-mockitoやばい
- 黒魔術を駆使したLibは避けよう
- JavaLibは無難
- sbtと向き合おう
