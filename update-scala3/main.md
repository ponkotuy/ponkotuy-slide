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

↓

Scala3移行をブロックしているLib・コードが明らかに

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


- `mock[Hoge]` => `mock(classOf[Hoge])`
- `hoge.func returns fuga` => `when(hoge.func).thenReturn(fuga)`
- `there was one(hoge.func) => verify(hoge, times(1)).func`


## PR(remove specs2 mock)
Files changed(86) +1,346 -1,466

つらい


## circe-generic-extras
- ScalaのJSONライブラリcirce
- circeのdecode/encodeに便利な機能を追加する
- 内部実装はほぼマクロ


## 社内のcirce-generic-extras
decoderがデフォルト値を使うようになる

`case class Person(name: String = "default")`

デフォルトだと"{}"渡すとエラーになるが、通るように

**POSTのbodyパーサにcirce使ってるって…こと…？**

Play Forms使え案件


## circe-generic-extras依存削除
- 半分はdecoderを使ってないので純粋に削除
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
