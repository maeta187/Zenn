---
title: 'Next.js+TailwindCSSでライブラリを使用せずトースト機能を実装してみる'
emoji: '🍞'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: [Nextjs, React, TailwindCSS, TypeScript]
published: false
---

## はじめに

実務で新プロジェクトのでトースト機能が要件として出ました。

POC として、Nuxt.js 環境でライブラリを使用せず実装したので、今回は Next.js 環境で実装してみます。

CSS は TailwindCSS を使用して自分では CSS を一切書かないようにしてみました。

## 開発環境

- Next.js 13.4.19
- tailwindcss 3.3.3
- heroicons 2.0.18

## Context の実装

トーストは主に API のレスポンスを表示するため、複数ページで使用することが予想されます。
表示ロジックコンポーネントで持つとページごとにコンポーネントの配置、Props の受け渡しを行う必要があります。
都度実装するのは面倒なので、Context を使用してグローバルに管理することにしました。

```tsx:src/context/ToastContext.tsx
'use client'

import { createContext, useContext, useState, useEffect, useRef } from 'react'
import Toast from '@/app/_components/Toast'

type ToastType = 'success' | 'warning' | 'error'

interface ToastContext {
  showToast: (message: string, type: ToastType) => void
  closeToast: () => void
}

export const ToastContext = createContext<ToastContext>({
  showToast: () => {},
  closeToast: () => {}
})

export const useToast = () => {
  return useContext(ToastContext)
}

export const ToastProvider = ({ children }: { children: React.ReactNode }) => {
  const [toastMessage, setToastMessage] = useState<string>('')
  const [toastType, setToastType] = useState<ToastType>('success')
  const [isShowToast, setShowToast] = useState<boolean>(false)
  const timer = useRef<ReturnType<typeof setTimeout>>()

  const showToast = (message: string, type: ToastType) => {
    // すでに実行されているsetTimeout()をキャンセルする
    clearTimeout(timer.current)
    setToastMessage(message)
    setToastType(type)
    setShowToast(true)
  }

  const closeToast = () => {
    // すでに実行されているsetTimeout()をキャンセルする
    clearTimeout(timer.current)
    setShowToast(false)
  }

  useEffect(() => {
    if (!isShowToast) return
    // 5秒後にToastを非表示にする
    timer.current = setTimeout(() => {
      setShowToast(false)
    }, 5000)
  }, [isShowToast])

  return (
    <ToastContext.Provider value={{ showToast, closeToast }}>
      {isShowToast && <Toast message={toastMessage} toastType={toastType} />}
      {children}
    </ToastContext.Provider>
  )
}
```

`_components`からコンポーネントをインポートしていますがそれについては後ほど説明します。
トースト内に表示テキスト、トーストの種類、表示/非表示を管理する値を状態管理します。
`showToast`、`closeToast` は Context から呼び出す関数です。
それぞれ、トーストを表示する、トーストを非表示にする関数です。
`useState`の Dispatch をエクポートして実行する方法もあると思いますが、表示/非表示の役割を持った関数を Context から呼び出す方がシンプルなのでこのように実装しました。

## トーストのコンポーネントの実装

`ToastContext.tsx`内でインポートしているトーストコンポーネントの実装です。

```tsx:src/app/_components/Toast.tsx
import { useToast } from '@/context/ToastProvider'

type ToastProps = {
  message: string
  toastType: 'success' | 'warning' | 'error'
}

const SuccessToast = ({
  message,
  onCloseTost
}: {
  message: string
  onCloseTost: () => void
}) => {
  return (
    <div
      className='
        animate-fade-out absolute top-5 left-1/2 -translate-x-2/4 w-72 px-2 py-4 rounded-xl flex
        text-green-700 bg-green-200 border-green-700
      '
    >
      <div>
        <svg
          xmlns='http://www.w3.org/2000/svg'
          fill='none'
          viewBox='0 0 24 24'
          strokeWidth={1.5}
          stroke='currentColor'
          className='w-6 h-6'
        >
          <path
            strokeLinecap='round'
            strokeLinejoin='round'
            d='M9 12.75L11.25 15 15 9.75m-3-7.036A11.959 11.959 0 013.598 6 11.99 11.99 0 003 9.749c0 5.592 3.824 10.29 9 11.623 5.176-1.332 9-6.03 9-11.622 0-1.31-.21-2.571-.598-3.751h-.152c-3.196 0-6.1-1.248-8.25-3.285z'
          />
        </svg>
      </div>
      <p className='pl-1'>{message}</p>
      <div className='ml-auto cursor-pointer' onClick={onCloseTost}>
        <svg
          xmlns='http://www.w3.org/2000/svg'
          fill='none'
          viewBox='0 0 24 24'
          strokeWidth={1.5}
          stroke='currentColor'
          className='w-6 h-6'
        >
          <path
            strokeLinecap='round'
            strokeLinejoin='round'
            d='M6 18L18 6M6 6l12 12'
          />
        </svg>
      </div>
    </div>
  )
}

const WarningToast = ({
  message,
  onCloseTost
}: {
  message: string
  onCloseTost: () => void
}) => {
  return (
    <div
      className='
        animate-fade-out absolute top-5 left-1/2 -translate-x-2/4 w-72 px-2 py-4 rounded-xl flex
        text-yellow-700 bg-yellow-200 border-yellow-700
      '
    >
      <div>
        <svg
          xmlns='http://www.w3.org/2000/svg'
          fill='none'
          viewBox='0 0 24 24'
          strokeWidth={1.5}
          stroke='currentColor'
          className='w-6 h-6'
        >
          <path
            strokeLinecap='round'
            strokeLinejoin='round'
            d='M12 18v-5.25m0 0a6.01 6.01 0 001.5-.189m-1.5.189a6.01 6.01 0 01-1.5-.189m3.75 7.478a12.06 12.06 0 01-4.5 0m3.75 2.383a14.406 14.406 0 01-3 0M14.25 18v-.192c0-.983.658-1.823 1.508-2.316a7.5 7.5 0 10-7.517 0c.85.493 1.509 1.333 1.509 2.316V18'
          />
        </svg>
      </div>
      <p className='pl-1'>{message}</p>
      <div className='ml-auto cursor-pointer' onClick={onCloseTost}>
        <svg
          xmlns='http://www.w3.org/2000/svg'
          fill='none'
          viewBox='0 0 24 24'
          strokeWidth={1.5}
          stroke='currentColor'
          className='w-6 h-6'
        >
          <path
            strokeLinecap='round'
            strokeLinejoin='round'
            d='M6 18L18 6M6 6l12 12'
          />
        </svg>
      </div>
    </div>
  )
}

const ErrorToast = ({
  message,
  onCloseTost
}: {
  message: string
  onCloseTost: () => void
}) => {
  return (
    <div
      className='
        animate-fade-out absolute top-5 left-1/2 -translate-x-2/4 w-72 px-2 py-4 rounded-xl flex
        text-red-700 bg-red-200 border-red-700'
    >
      <div>
        <svg
          xmlns='http://www.w3.org/2000/svg'
          fill='none'
          viewBox='0 0 24 24'
          strokeWidth={1.5}
          stroke='currentColor'
          className='w-6 h-6'
        >
          <path
            strokeLinecap='round'
            strokeLinejoin='round'
            d='M12 9v3.75m0-10.036A11.959 11.959 0 013.598 6 11.99 11.99 0 003 9.75c0 5.592 3.824 10.29 9 11.622 5.176-1.332 9-6.03 9-11.622 0-1.31-.21-2.57-.598-3.75h-.152c-3.196 0-6.1-1.249-8.25-3.286zm0 13.036h.008v.008H12v-.008z'
          />
        </svg>
      </div>
      <p className='pl-1'>{message}</p>
      <div className='ml-auto cursor-pointer' onClick={onCloseTost}>
        <svg
          xmlns='http://www.w3.org/2000/svg'
          fill='none'
          viewBox='0 0 24 24'
          strokeWidth={1.5}
          stroke='currentColor'
          className='w-6 h-6'
        >
          <path
            strokeLinecap='round'
            strokeLinejoin='round'
            d='M6 18L18 6M6 6l12 12'
          />
        </svg>
      </div>
    </div>
  )
}

const Toast = ({ message, toastType }: ToastProps) => {
  const { closeToast } = useToast()

  const handleCloseToast = () => {
    closeToast()
  }

  switch (toastType) {
    case 'success':
      return (
        <div className='relative'>
          <SuccessToast message={message} onCloseTost={handleCloseToast} />
        </div>
      )
    case 'warning':
      return (
        <div className='relative'>
          <WarningToast message={message} onCloseTost={handleCloseToast} />
        </div>
      )
    case 'error':
      return (
        <div className='relative'>
          <ErrorToast message={message} onCloseTost={handleCloseToast} />
        </div>
      )
  }
}

export default Toast
```

`success`、`warning`、`error` の 3 種類のトーストコンポーネントを用意しました。
それを Context から受け取った`toastType`によって表示するトーストを切り替えています。
`SuccessToast`、`WarningToast`、`ErrorToast`のコンポーネント内でアイコンを表示していますが、`Heroicons`を使用しています。
`Heroicons`の開発元は TailwindCSS と同じ開発チームなので統一感を出すために使用しました。

https://heroicons.com/

## アニメーションの実装

トーストを表示する際よくあるのが上から下にスライドして表示するアニメーションや、フェードインするアニメーションです。
今回はフェードインするアニメーションでトーストを表示することにします。
それぞれのトーストコンポーネントに`animate-fade-out`というクラスを付与しています。
ですが、このクラスは TailwindCSS には存在しないクラスです。
このためだけに CSS を書くのは面倒です…
公式ドキュメントにこのような記述がありました。

https://v1.tailwindcss.com/docs/animation#customizing

`tailwind.config.js`で設定すれば自分好みのアニメーションを作成できるようです。
これを元にアニメーションの設定を行います。

```js:tailwind.config.js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: ['./src/**/*.{js,ts,jsx,tsx,mdx}'],
  theme: {
    extend: {
      animation: {
        'fade-out': 'fade-out 5s ease both'
      },
      keyframes: {
        'fade-out': {
          from: {
            opacity: '1'
          },
          to: {
            opacity: '0'
          }
        }
      }
    }
  },
  plugins: []
}
```

設定はできましたが、コンポーネントで使用するためにはどうすればいいのか分かりませんでした。
調べてみると参考になる記事がありました。

https://zenn.dev/angelecho/articles/f171ca2b3b1f6a

どうやら`animate-`＋`keyframes`のプロパティ名でアニメーションを使用できるようです。

![CSS](/images/toast-1.png)

TailwindCSS のプロパティが書かれている CSS ファイルに`animate-fade-out`が追加されていることが確認できました。
問題無ければアニメーションが効くはずなのでその辺りもこの後の動作確認で確認していきます。

## 動作確認

諸々の実装が完了したので動作確認を行います。
今回は Button をクリックした際にトーストを表示するようにして確認していきます。
最小限の確認のため`src/app/toast/page.tsx`に`ToastProvider`をインポートして確認します。

```tsx:src/app/toast/page.tsx
import ToastContent from '@/app/toast/ToastContent'
import { ToastProvider } from '@/context/ToastProvider'

export default function Page() {
  return (
    <ToastProvider>
      <ToastContent />
    </ToastProvider>
  )
}
```

次に`ToastContent.tsx`の作成です。
押したボタンによってトーストの種類を変えています。
closeToast ボタンを押すとトーストが非表示になるようにしています。

```tsx:src/app/toast/ToastContent.tsx
'use client'

import { useToast } from '@/context/ToastProvider'

const ToastContent = () => {
  const { showToast, closeToast } = useToast()

  return (
    <div>
      <div className='text-center w-9/12 my-0 mx-auto pt-36'>
        <button
          className='p-2 border-2 rounded-xl text-green-700 bg-green-200 border-green-700'
          type='button'
          onClick={() => showToast('success', 'success')}
        >
          success
        </button>
        <button
          className='ml-2 p-2 border-2 rounded-xl text-yellow-700 bg-yellow-200 border-yellow-700'
          type='button'
          onClick={() => showToast('warning', 'warning')}
        >
          warning
        </button>
        <button
          className='ml-2 p-2 border-2 rounded-xl text-red-700 bg-red-200 border-red-700'
          type='button'
          onClick={() => showToast('error', 'error')}
        >
          error
        </button>
        <button
          className='ml-2 p-2 border-2 rounded-xl text-gray-700 bg-gray-200 border-gray-700'
          type='button'
          onClick={closeToast}
        >
          closeToast
        </button>
      </div>
    </div>
  )
}

export default ToastContent
```

ブラウザで確認したものがこちらです。
※gif が重いのでもっさりしていますがご了承ください。
![CSS](/images/toast-movie.gif)

無事動作が確認できました！
トーストの出し分け、アニメーション、closeToast が正常に動作していることが確認できました。

## 課題

### ページ遷移時にトーストの表示が消える

ページ遷移時にトーストの表示が消えてしまいます。
Layout に配置しても解決しなかったので調査が必要です。

## まとめ

これが絶対正解ではないと思いますが、自分なりに実装してみました。
ライブラリを使用すればもっと簡単に実装できますが、実務だと使いたいライブラリが必ず使えるとは限りません。
実際作ってみると、トースト 1 つとっても考慮することが多く、実装の中で勉強になりました。
Nuxt.js で実装したものも考慮漏れがありそうなので見返してみます。
間違いや、改善点があればご指摘いただけると幸いです。

## 参考

https://heroicons.com/

https://v1.tailwindcss.com/docs/animation#customizing

https://zenn.dev/angelecho/articles/f171ca2b3b1f6a

**[React] 副作用フックを使用して setTimeout のタイマーをリセットする**

https://dev.classmethod.jp/articles/reset-timer-for-settimeout-using-use-effect-in-react/
