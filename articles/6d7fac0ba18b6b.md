---
title: "Compose Multiplatform でも使える高機能画像表示の神ライブラリ ZoomImage"
emoji: "🌁"
type: "tech"
topics:
  - "android"
  - "kotlin"
  - "compose"
  - "kmp"
  - "cmp"
published: true
published_at: "2024-12-24 07:00"
publication_name: "aldagram_tech"
---

こんにちは！[アルダグラム](https://aldagram.com/about/)でエンジニアをしている渡邊です。

弊社の KANNA のモバイルアプリでは、以前から画像に対して線やテキストを書き込み、書き込んだ内容を新しい画像として出力することができるという、**画像編集機能**があります。

画像編集時には画像を回転することができなかったのですが、先日、画像も回転させられるようにする機能拡張を行いました。

以下はそのサンプル動画です。

![](https://storage.googleapis.com/zenn-user-upload/5247ded608f1-20241220.gif =300x)

以前までは画像編集時には [TouchImageView](https://github.com/MikeOrtiz/TouchImageView) というライブラリを使って画像を表示していたのですが、このライブラリには画像を回転させる機能がありませんでした。

そのため回転機能がある画像表示ライブラリを探していたのですが、 [ZoomImage](https://github.com/panpf/zoomimage) というライブラリがかなり神がかっていた（注: 筆者の個人的な意見です）ので、今回はこのライブラリの紹介をしたいと思います。

※ ちなみにこのライブラリはチームメンバーの方が見つけてくださいました

https://github.com/panpf/zoomimage

# ZoomImage の主な特徴

- Compose と Android View の両方サポートされている
- Compose Multiplatform 対応されている
- 画像の拡大縮小や移動の基本的な機能に加え、90度単位の画像回転が行える
- 画像のスケール値、移動量、回転角度の取得方法がイケてる（※ 筆者の個人的な意見です）
- サブサンプリングのサポート
- 複数の画像ローダーのサポート
    - [Sketch](https://github.com/panpf/sketch)、[Coil](https://github.com/coil-kt/coil)、[Glide](https://github.com/bumptech/glide)、[Picasso](https://github.com/square/picasso)
- スクロールバーの表示


# Compose と Android View の両方サポートされている

Compose も Android View も両方サポートされています。 元々使っていた画像表示ライブラリは Android View で作られていたため、Android View のサポートがあるのは非常に助かりました。

# Compose Multiplatform 対応されている

さらに Compose Multiplatform 対応されています。 Android だけでなく、iOS、デスクトップアプリ、Web でも使えます。

# 画像の拡大縮小や移動の基本的な機能に加え、90度単位の画像回転が行える

画像の拡大縮小や移動を行える画像表示ライブラリは多いですが、回転もサポートしているライブラリはそんなに多くないと思います。 このライブラリでは90度単位で画像回転が行えちゃいます。

# 画像のスケール値、移動量、回転角度の取得方法がイケてる

`Transform` という型に画像のスケール値や移動量、回転角度が保持されます。この `Transform` のインスタンスは `ZoomableState` を経由することで取得できますが、以下の3つのパターンがあります。

```kotlin
val zoomState: ZoomState by rememberZoomState()
val zoomable: ZoomableState = zoomState.zoomable

// Transform 型のプロパティを取得する際に以下の3つのパターンがある
zoomable.baseTransform
zoomable.userTransform
zoomable.transform
```

これらの3つの違いは以下のようになっています。

- `baseTransform`
    - オリジナルの画像を画面に初期表示する際に使用された変換値
    - 例えばオリジナルの画像サイズが2000x1000で画面に初期表示されるサイズが1000x500だった場合、スケール値は0.5になる
- `userTransform`
    - `baseTransform` からユーザーが画像を拡大・移動・回転した際の変換値
    - 例えば画面に表示される画像サイズが1000x500でユーザーが画像を拡大表示した際のサイズが2500x1250だった場合、スケール値は2.5になる
- `transform`
    - `baseTransform` と `userTransform` を合成した値
    - 例えば上記の例を合成すると、この時のスケール値は1.25になる

これらのプロパティによって、用途に応じた値をそれぞれ取得することが可能になります。

また上記のプロパティは Compose の `State<Transform>` 型になっているので、値の変更を監視することができます。

さらに Android View では上記のプロパティは Kotlin Coroutines の `StateFlow<TransformCompat>` 型になっており、同様に値の変更を監視することが可能になっています。

その他にも様々なプロパティが用意されていますので、詳細は[ドキュメント](https://github.com/panpf/zoomimage/blob/main/docs/wiki/getstarted.md#public-properties)を参考にしてください。


# サブサンプリングのサポート

サイズの大きい画像を読み込むと、メモリが足りずにアプリがクラッシュしてしまう可能性があります。 そのため画像ローダーのライブラリは画素データを間引いてロードしますが、そうすると画像のサイズは小さくなってしまい、表示される画像がぼやけてしまいます。

そのため ZoomImage では画像を拡大した時の画面にはオリジナルの画像の断片を表示させることができます。

これによってアプリがクラッシュすることがなく、画像を鮮明に表示させることが可能です。

実装については[ドキュメント](https://github.com/panpf/zoomimage/blob/main/docs/wiki/subsampling.md
)を参考にしてください。

# 複数の画像ローダーのサポート

前述のサブサンプリングのために、以下の複数の画像ローダーをサポートしています。

- [Sketch](https://github.com/panpf/sketch)
- [Coil](https://github.com/coil-kt/coil)
- [Glide](https://github.com/bumptech/glide)
- [Picasso](https://github.com/square/picasso)

Sketch は私は知らなかったのですが、ZoomImage のライブラリの作者が開発している画像ローダーのようです。

# スクロールバーの表示

スクロールバーを表示することができます。

また、スクロールバーの色やサイズなどもカスタマイズ可能です。

![](https://storage.googleapis.com/zenn-user-upload/22b31a3e5986-20241220.gif =300x)

# 最後に

ZoomImage というライブラリについて紹介させていただきました。 私は Compose は使ってなく Android View のみを使ったのですが、所感としては高機能で非常に使いやすかったです。

今回紹介した内容以外にもいくつか機能がありますので、気になる方はぜひチェックしてみてください！

この記事がどなたかの参考になれば幸いです。

もっとアルダグラムエンジニア組織を知りたい人、ぜひ下記の情報をチェックしてみてください！