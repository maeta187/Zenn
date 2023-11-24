---
title: '個人開発アプリでdaisyUIを採用した話'
emoji: '🌻'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: [Nextjs, React, TailwindCSS, daisyUI]
published: 2023-11-24 20:00
---

## はじめに

数ヶ月前から個人開発をしています。
CSS フレームワークで Tailwind CSS, daisyUI を採用したことをまとめてみました！

## 開発環境

- Next.js 13.4.19
- TailwindCSS 3.3.3
- daisyUI 3.7.7

## フレームワークの調査基準

今回フレームワークを選定するに辺り、基準として以下を軸にして考えました。

- App Router に対応しているか
- CSS を記述するコスト
- UI パーツが提供されているか
- ReactHookForm との相性

以上の 4 点を軸に調査、選定を進めていきました。

## App Router に対応しているか

当時は App Router に対応しているものが少なく、Tailwind CSS 意外だと Panda CSS が対応しているぐらいでした。
この 2 つを元に**CSS を記述するコスト**と**UI パーツが提供されているか**の調査を進めていきました。

## CSS を記述するコスト

**Tailwind CSS**
クラス名を追加することでスタイルリングが可能で CSS を記述するコストはほぼ 0 に近いです。
アニメーションなどは設定する必要がありますが、1 度分かってしまえば実装は簡単です。
アニメーションについては過去に自分の記事でも触れているので参考にして見てくだいさい。

https://zenn.dev/pe_be_o/articles/maeta-187-articles_303daee8db0353

**Panda CSS**
Panda CSS は CSS in JS なので下記のような CSS プロパティを記述する方法になります。
※コードは公式ドキュメントから参照しました。

```tsx
// className内にスタイルを記述するパテーン
import { css } from '../styled-system/css'

function App() {
  return <div className={css({ br: 'sm' })} />
}

// JSXのPropsを使用したバージョン
import { styled } from '../styled-system/jsx'

function App() {
  return <styled.div br='sm' />
}
```

Panda CSS は自分でプロパティーを記述するのでややコストがかかる模様。
また、過去に styled-components 使ったことがありますが、少し苦手意識があったので CSS in JS に抵抗感を持っていました。

## UI パーツが提供されているか

**Tailwind CSS**
Tailwind CSS 単体では提供されていませんが、daisyUI の様なライブラリを使用すれば UI パーツも提供される。

**Panda CSS**
UI パーツの提供はなく自分で Preset や Utilities を作る必要がある。
この時点で自分の中で軍配が Tailwind CSS に上りました。

**ReactHookForm との相性**に入る前に Tailwind CSS と併せて使用するライブラリを調査しました。

## Tailwind CSS ベースの UI フレームワークの調査と選定

色々調べてみて最終的な有力候補は以下の 3 つとなりました。

### Flowrift

https://flowrift.com/c/banner

**メリット**

- Tailwind CSS のクラスのみで作られているので追加のライブラリのインストールなどのセッティング不要で使える
- ページごとに提供されているのでデザイン不要

**デメリット**

- ページごとに提供されているので特定のコンポーネントを使う場合は切り出す必要がある
- コンポーネント単位で提供されていないのでローディングなど言ったアニメーションのあるコンポーネントは提供されていない
- ダークモード未対応

### daisyUI

https://daisyui.com/

**メリット**

- クラスベースでコンポーネントが提供されている
- テーマはデフォルトで設定されているが自分でカスタムすることも可能
- ダークモード対応
- 公式ドキュメントが日本語にも対応

**デメリット**

- Tailwind CSS + daisyUI のクラス名を書くので`className`の記述が膨大になるケースがある

### NextUI

**メリット**

https://nextui.org/

- Tailwind CSS の開発チームが開発しているので親和性が高い
- レイアウトやカラーなどのカスタマイズ性が高い
- JSX コンポーネントで提供されているので`className`の記述が少ない
- ダークモード対応

**デメリット**

- React ベースのコンポーネントなので Vue.js などでは使用できない
- トップレベルの Layout に Provider を設定する必要があるのでレンダリングに影響があるかも?

これを踏まえて daisyUI と NextUI が最有力候補となりました。

## ReactHookForm との相性

個人開発ではサインイン、ログインなどのフォームを作成する予定だったので ReactHookForm との相性も重要なポイントでした。
ReactHookForm は自分もあまりちゃんと使ったことがなかったのでその学習も兼ねて今回採用することにしました。
そして、daisyUI と NextUI を使用した上で ReactHookForm との相性を調査を行いました。

### daisyUI

こちらはクラスベースでスタイリングするので ReactHookForm と動作を阻害することは無く、問題なく使用できました。

### NextUI

こちらは JSX コンポーネントでスタイリングするので ReactHookForm との相性はある条件を満たせば問題なく使用できました。
まずはこのようなコードを書いてみます。

```tsx:Form.tsx
'use client'

import { SubmitHandler, useForm } from 'react-hook-form'
import { Button } from '@nextui-org/react'
import { Input } from '@nextui-org/input'

const Form = () => {
  const handleOnSubmit: SubmitHandler<ContactType> = (data) => {
    console.log(data)
  }

  const {
    register,
    handleSubmit,
    formState: { errors: formatError, isValid, isSubmitting }
  } = useForm<ContactType>({
    mode: 'onBlur',
    defaultValues: {
      telephone: ''
    },
    resolver: valibotResolver(ContactSchema)
  })

return (
  <form
    method='post'
    className='flex flex-col space-y-10'
    onSubmit={(event) => {
      handleSubmit(handleOnSubmit)(event)
    }}
  >
  <div className='flex flex-col space-y-1'>
    <Input
      key='secondary'
      color='secondary'
      type='text'
      label='Tel'
      className=''
      {...register('telephone')}
    />
  </div>
  <div>
    <Button
      type='submit'
      disabled={!isValid || isSubmitting}
      color='secondary'
      variant='bordered'
    >
      submit
    </Button>
</div>
</from>
)

export default Form
```

これで入力するとブラウザのコンソールにエラーが発生します。

![BrowserError](/images/react-hook-from-ui-1.jpg)

エラー内容をざっくり説明すると、コンポーネントが uncontrolled から controlled に変更され、未定義の値が定義済みの値に変更されたという内容です。
`useForm`内の`defaultValues: undefined,`にデフォルト値を設定することでエラーは解消されようです。
なので下記のように修正します。

```tsx
const {
  register,
  handleSubmit,
  formState: { errors: formatError, isValid, isSubmitting }
} = useForm<ContactType>({
  mode: 'onBlur',
  defaultValues: {
    // ここにデフォルト値を設定
    telephone: ''
  },
  resolver: valibotResolver(ContactSchema)
})
```

フォームは複数の入力項目があるので、オブジェクトでデフォルト値を設定しました。
これで解決したはず...解決できないだと...？
自分でも解決方法が浮かばなかったので GitHub Copilot に聞いてみました。
そして返ってきた回答を要約すると以下のような内容でした。

- NextUI の Input コンポーネントで React Hook Form の register 関数を使用している
- NextUI の Input コンポーネントはカスタムコンポーネントで、内部で input 要素をラップしている
- したがって、register 関数を直接適用すると期待通りに動作しない可能性がある
- register 関数を適用するために、Controller コンポーネントを使用することを検討してみてくださいとのこと
- Controller コンポーネントは、React Hook Form と他の UI ライブラリの間の結合するために使用するとのこと

ということで、Controller コンポーネントを使用した方法に修正してみました。
修正したコードがこちらです。

```tsx:Form.tsx
'use client'

import { SubmitHandler, useForm, Controller } from 'react-hook-form'
import { Button } from '@nextui-org/react'
import { Input } from '@nextui-org/input'

const Form = () => {
  const handleOnSubmit: SubmitHandler<ContactType> = (data) => {
    console.log(data)
  }

  const {
    register,
    handleSubmit,
    control,
    formState: { errors: formatError, isValid, isSubmitting }
  } = useForm<ContactType>({
    mode: 'onBlur',
    defaultValues: {
      telephone: ''
    },
    resolver: valibotResolver(ContactSchema)
  })

  return (
    <form
      method='post'
      className='flex flex-col space-y-10'
      onSubmit={(event) => {
        handleSubmit(handleOnSubmit)(event)
      }}
    >
      <div className='flex flex-col space-y-1'>
        <Controller
          name='telephone'
          control={control}
          defaultValue=''
          render={({ field }) => (
            <Input
              key='secondary'
              color='secondary'
              type='text'
              label='Tel'
              {...field}
            />
          )}
        />
      </div>
      <div>
        <Button
          type='submit'
          disabled={!isValid || isSubmitting}
          color='secondary'
          variant='bordered'
        >
          Bordered
        </Button>
      </div>
    </form>
  )
}

export default Form
```

はい、これでエラーは解消されました。
ですが、ここまで実装してまで NextUI を使用するメリットはあるのか？という疑問が残りました。
また、個人開発の雰囲気をポップにしたいという思いもあり、daisyUI を採用することにしました。

## daisyUI を使った感想

daisyUI を使ってみての感想ですが、daisyUI のデメリットでも挙げた Tailwind CSS のクラス名を書くので`className`の記述が膨大になるケースがあります。
膨大になるのは防ぐことはできませんが、順番を決めることで統一感を出すことはできます。
**eslint-plugin-tailwindcss**を使用すれば、クラス名の順番を強制することができます。
それを除けば特にストレスや開発が止まると言ったことは無いです。
daisyUI をインストールするとデフォルトでターミナル上にメッセージが表示されます。
設定で非表示にはできますが、エラーが発生した時にも影響があるので非表示にはしていません。
daisyUI のコンポーネントはシンプルにまとまっているので、コンポーネントのカスタマイズもしやすいです。
また、候補で挙げた Flowrift もクラスベースなのでこちらと併用もでき Flowrift から UI のサンプルをコピペして daisyUI でスタイリングすることもできます。
採用して良かったです。

## まとめ

今回は個人開発アプリで daisyUI を採用した話をまとめてみました。
CSS フレームワークも今では膨大な数があり、Tailwind CSS ベースで見てもかなりの数がありました。
なので自分の開発環境に合ったものを選定することが大切だと感じました。
また、当時は CSS in JS のフレームワークは Panda CSS のみが対応していましたが、今では Emotion や Chakra UI なども App Router に対応しました。
CSS in JS に抵抗感を克服したいのでそのうち使ってみたいと思います。

## 参考

https://flowrift.com/c/banner

https://daisyui.com/

https://nextui.org/

https://react-hook-form.com/

https://zenn.dev/buzzkuri_tech/articles/198016c6407f70

https://zenn.dev/brachio_takumi/articles/a8fecd8b1b2742

https://zenn.dev/pe_be_o/articles/maeta-187-articles_303daee8db0353
