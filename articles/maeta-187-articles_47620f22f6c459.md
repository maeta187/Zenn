---
title: 'Next/Imageを何となく使ってたらボコボコにされた'
emoji: '🤕'
type: 'tech'
topics: ['Nextjs', 'React', 'Image']
published: false
---

## はじめに

Next.js を使用して LP を制作した際の失敗談です。
知識不足の戒めとして、記事に残しておきます。

:::message
筆者は React/Next.js の実務経験がないので独学でまとめています。
もし、間違いや誤解があればご指摘いただけると幸いです。
:::

## 開発環境

- Next.js 13.4.19

## 事象

Next/Image を使用して画像を表示させ、`width`/`height`を Figma で確認した値を直接指定して実装進めていました。

勿論、レスポンシブ対応も事前に分かっていましたが、一旦 PC サイズで実装してから、スマホサイズでの調整を行う予定でした。

ところが、`width`/`height`を固定値にしていたためレスポンシブ対応ができない状況に陥ってしまいました。

調べたり、試行錯誤して出来上がったのが以下のコードです。(完全再現とまではいかないですがおおよそこんな感じです)

```tsx:ImageComponent
import Image from 'next/image'
import styles from '@/app/sample/image/styles.module.css'

const ImageComponent = () => {
  return (
    <div className={styles.inner}>
      <div className={styles.container}>
        <Image
          src='https://picsum.photos/id/237/200/300'
          alt='image-sample'
          width={476}
          height={323}
          style={{ width: '100%', height: 'auto', objectFit: 'contain' }}
        />
      </div>
    </div>
  )
}

export default ImageComponent
```

```css:styles.module.css
@charset "utf-8";

.inner {
  max-width: 1152px;
  width: 61.622rem;
  margin: 0 auto;
}

.container {
  width: 300px;
}

@media screen and (max-width: 768px) {
  .inner {
    max-width: 768px;
    padding-left: 4%;
    padding-right: 4%;
  }
}

@media screen and (max-width: 425px) {
  .inner {
    max-width: 425px;
    width: 100%;
  }
}
```

切羽詰まっていたんでしょうね。`width`/`height`を固定値にしているにも関わらず、`style`でも指定すると言う無茶なことをしています。

サンプル画像で使用したワンちゃんもどこか悲しそうに見えます。

## 解決方法

### レスポンシブ

まずは公式ドキュメントを確認します。

きちんとレスポンシブについて触れていますね！

@[card](https://nextjs.org/docs/app/building-your-application/optimizing/images#responsive)

しかもサンプルコードのおまけ付きです。

では一旦これをコピペしてみましょう。

ん?何やらエラーが出ていますね。

```bash
Error: Image with src "https://picsum.photos/id/237/200/300" is missing required "width" property.
```

エラー内容を読むと、`width`が必須とのことです。

これは Next/Image の仕様なので仕方ありません。

ただし、`fill`を使用する場合は`width`/`height`を指定する必要はないようです。

では、`fill`を使用してみましょう。公式ドキュメントにも書いてありますが、`fill`を使用する場

合は親要素に`position: relative`と`display: block`を指定する必要があります。

それを踏まえて、以下のようにコードを書き換えます。

```tsx:ImageComponent
import Image from 'next/image'

const ImageComponent = () => {
  return (
    <div style={{ display: 'block', position: 'relative' }}>
      <Image
        src='https://picsum.photos/id/237/200/300'
        alt='image-sample'
        sizes='100vw'
        fill
        style={{
          width: '100%',
          height: 'auto'
        }}
      />
    </div>
  )
}

export default ImageComponent
```

また、エラーが出ていますね…

```bash
Error: Image with src "https://picsum.photos/id/237/200/300" has both "fill" and "style.height" properties. Images with "fill" always use height 100% - it cannot be modified.
```

エラー内容を読むと、`fill`と`style.height`を同時に指定することはできないようです。
`fill`を指定すると`height`は自動的に`100%`になるため、`style.height`を指定することはできないようです。
では、`style.height`を削除してみましょう。
エラーは消えましたが、画像が表示されていませんね。
これは`fill`を指定すると`width`/`height`が`100%`になるため、画像自身はコンテンツサイズを持っておらず、親要素か兄弟要素に`width`/`height`を指定する必要があるようです。
兄弟要素がテキストだった場合、`width`/`height`を指定することは現実的ではないので、親要素に`width`/`height`を指定するのが良さそうです。
それを踏まえたコードが以下になります。

```tsx:ImageComponent
import Image from 'next/image'

const ImageComponent = () => {
  return (
    <div
      style={{
        display: 'block',
        position: 'relative',
        width: '400px',
        height: '600px'
      }}
    >
      <Image
        alt='image-sample'
        src='https://picsum.photos/id/237/200/300'
        sizes='100vw'
        fill
        style={{
          width: '100%'
        }}
      />
    </div>
  )
}

export default ImageComponent
```

![犬の画像](/images/next-image-1.png)

無事に画像が表示されました！
ワンちゃんも愛らしいです。
デベロッパーツールで画面幅が 200px 以下になった際も、自然に画像が縮小されていることが確認できます。

### 補足

今回は画像のみ表示させましたが、カードコンテンツのような場合、どこにテキストコンテンツを配置するのかが問題になります。
画像と兄弟要素とした場合、画像の下にテキストコンテンツが来てしまいます。
なので`width`/`height`を指定した親要素と兄弟要素にする必要があります。

```tsx:ImageComponent
import Image from 'next/image'

const ImageComponent = () => {
return (
  <div
    style={{
      display: 'flex',
      justifyContent: 'center',
      width: '1100px',
      margin: '0 auto'
    }}
  >
    <p>foo</p>
    <div
      style={{
        display: 'block',
        position: 'relative',
        width: '200px',
        height: '300px'
      }}
    >
      <Image
        alt='image-sample'
        src='https://picsum.photos/id/237/200/300'
        sizes='100vw'
        fill
        style={{
          width: '100%'
        }}
      />
    </div>
  </div>
)
}

export default ImageComponent
```

![犬とテキストの横並び](/images/next-image-2.png)

画像とテキストが横並びになりました。
これでカードコンテンツも実装できそうです！

## まとめ

改めて自分への戒めとしてまとめてみました。
作業していた時は 12 系の情報が多く、13 系で廃止されたプロパティがあることを知らなかったため、良い経験になりました。
おかげ様で、Next/Image のレスポンシブ対応については理解が深まり仲良くなれた気がします。
また、同じようにつまづいている方の助けになれば幸いです。
