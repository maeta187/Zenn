---
title: 'TypeScriptの実行環境をBunとBiomeに移行してみた'
emoji: '🚚'
type: 'tech'
topics: ['TypeScript', 'Bun', 'Biome']
published: true
published_at: 2024-07-21 19:00
---

## はじめに

転職してからサーバーサイドの勉強や AWS の SAA の勉強をしていて、一旦一区切り着きました。(SAA は無事合格しました。)
TypeScript もかなりアップデートが入り、気づいたらバージョンは`5.5`でした。
ここのキャッチアップから始めようと思い、勉強用の実行環境を久しぶりに立ち上げふと思いました。
「環境だいぶ昔に作ったからパッケージのバージョン古いんじゃね？」
なので確認してみました。

![ターミナル](/images/migration-1.png)

かなり古かったので、勉強の前に環境をアップデートしようと思いました。

## 既存のパッケージのバージョンアップ

当初は既存のパッケージのバージョンアップで進めていたのですが、1 つ障壁にぶつかったのが**ESLint**です。
バージョンが`9.x`から仕様が変わり`.eslintrc.json`から`eslint.config.js`へ変更し、動作確認を行いました。
一見動いてそうかと思いきや見慣れない warning が表示されました。

> warning File ignored because no matching configuration was supplied

簡単に訳すと「一致する設定が提供されなかったため、ファイルは無視された」と言うことです。
調べた結果、スクリプトで書いていたディレクトリ指定を書き換えると解決しました。

```json
// アップデート前
$ "lint-fix": "eslint --cache --fix src/ && prettier --write src/"

// アップデート後
$ "lint-fix":　"eslint --cache --fix src/**/*.ts && prettier --write src/**/*.ts"
```

但し、`warn`としているルールが`error`扱いになるなど違和感を感じたのを覚えています。
また、TS ファイルを実行した時に CommonJS と ES Modules の違いによるエラーも発生しました。
引越しも控えている中でできるだけ早く環境を新しくしたのと、TS を実行するだけの環境なのでパーケージ依存による管理コストを減らしたいと思い移行することにしました。

## Bun と Biome への移行

### Bun

@[card](https://bun.sh/)

Biome に移行するなら実行環境もついでに変えたいと思い、Bun に移行することにしました。
Node.js だと`ts-node`を使用する必要がありました。
実行するだけの為に`ts-node`を入れておくのもなんだかなーと思っていたのと、上記でも書いたように CommonJS と ES Modules に振り回されたくないのでデフォルトで TypeScript を実行できる Bun に移行しました。
Bun 自体はインストールしていたので 1 度`package.json`を削除してから`bun init`を実行しました。(あんまり正しい方法では無さそう)
後は、TypeScript のインスールとスクリプトの修正で完了です。

### Biome

@[card](https://biomejs.dev/ja/)

Biome は以前から気になっていたので、ここで導入することにしました。
Bun にも対応しているので、Bun 環境でも問題なく使用できます。

#### インストール

公式ドキュメントに沿ってインストール、設定を行っていきます。
`biome.json`を作る先に`biome init`を実行するのですが、下記のようなエラーが発生しました。

> Error: Cannot find module '@biomejs/cli-darwin-x64/biome'

調べて見ると下記のページが見つかりました。

@[card](https://biomejs.dev/ja/guides/manual-installation/)

内容を確認すると、Node.js や npm を使用しない場合は Biome のスタンドアロンな CLI バイナリを使用できるとのことです。
自分は Bun を使用しているのでこちらの方法で進めていきます。
curl コマンド実行後、再度`biome init`を実行して無事`biome.json`が作成されました！

#### マイグレーション

Biome にはマイグレーションコマンドが提供されており、そちらを使用することで ESLint、Prettier の設定を簡単に移行することができます。

#### ESLint

```bash
$ biome migrate eslint --write
```

これを実行すると下記のようなエラーが発生しました。

> Migration has encountered an error: `node` was invoked to resolve '@typescript-eslint/eslint-plugin'. This invocation failed with the following error:
>
> Migration has encountered an error: `node` was invoked to resolve 'eslint-config-prettier'. This invocation failed with the following error:

移行前に使用していたパッケージがインストールされておらず、マイグレーションを行う際参照できないためエラーが発生しているようです。
表示されたパッケージをインストールし、再度マイグレーションを行うと無事移行できました。
マイグレーション時にインストールしたパッケージは恐らく不要なので削除しておきます。

#### Prettier

```bash
$ biome migrate prettier --write
```

こちらも ESLint と同様に実行するとマイグレーションがされるかと思いきや、下記のエラーが発生しました。

> Migration has encountered an error: `node` was invoked to resolve './prettier.config.js'. This invocation failed with the following error:

原因を調べてみましたが、解決策が見つからず諦めかけていた時でした。

![公式ドキュメント](/images/migration-2.png)

「っん？サンプルのファイル拡張子が`.json`だと…」
試しに拡張子を`.json`に変更して再度マイグレーションを実行すると…

> Prettier's `"endOfLine": "auto"` option is not supported in Biome. The default `"lf"` option is used instead.
> .prettierrc.json has been successfully migrated.

無事マイグレーションが完了しました！
マイグレーションを行う際、ESLint と Prettier のファイル拡張子は`.json`にする必要があるようです。

#### 動作確認

マイグレーションが完了した`biome.json`が以下のようになります。

:::details biome.json

```json
{
  "$schema": "https://biomejs.dev/schemas/1.8.3/schema.json",
  "vcs": {
    "clientKind": "git",
    "enabled": true,
    "useIgnoreFile": true,
    "defaultBranch": "master"
  },
  "formatter": {
    "enabled": true,
    "formatWithErrors": false,
    "indentStyle": "space",
    "indentWidth": 2,
    "lineEnding": "lf",
    "lineWidth": 80,
    "attributePosition": "auto"
  },
  "organizeImports": { "enabled": true },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": false,
      "complexity": {
        "noBannedTypes": "error",
        "noExtraBooleanCast": "error",
        "noMultipleSpacesInRegularExpressionLiterals": "error",
        "noUselessCatch": "error",
        "noUselessTypeConstraint": "error",
        "noWith": "error"
      },
      "correctness": {
        "noConstAssign": "error",
        "noConstantCondition": "error",
        "noEmptyCharacterClassInRegex": "error",
        "noEmptyPattern": "error",
        "noGlobalObjectCalls": "error",
        "noInvalidConstructorSuper": "error",
        "noInvalidNewBuiltin": "error",
        "noNonoctalDecimalEscape": "error",
        "noPrecisionLoss": "error",
        "noSelfAssign": "error",
        "noSetterReturn": "error",
        "noSwitchDeclarations": "error",
        "noUndeclaredVariables": "error",
        "noUnreachable": "error",
        "noUnreachableSuper": "error",
        "noUnsafeFinally": "error",
        "noUnsafeOptionalChaining": "error",
        "noUnusedLabels": "error",
        "noUnusedPrivateClassMembers": "error",
        "noUnusedVariables": "off",
        "useArrayLiterals": "off",
        "useIsNan": "error",
        "useValidForDirection": "error",
        "useYield": "error"
      },
      "style": {
        "noNamespace": "error",
        "useAsConstAssertion": "error",
        "useBlockStatements": "off"
      },
      "suspicious": {
        "noAsyncPromiseExecutor": "error",
        "noCatchAssign": "error",
        "noClassAssign": "error",
        "noCompareNegZero": "error",
        "noControlCharactersInRegex": "error",
        "noDebugger": "error",
        "noDuplicateCase": "error",
        "noDuplicateClassMembers": "error",
        "noDuplicateObjectKeys": "error",
        "noDuplicateParameters": "error",
        "noEmptyBlockStatements": "error",
        "noExplicitAny": "error",
        "noExtraNonNullAssertion": "error",
        "noFallthroughSwitchClause": "error",
        "noFunctionAssign": "error",
        "noGlobalAssign": "error",
        "noImportAssign": "error",
        "noMisleadingCharacterClass": "error",
        "noMisleadingInstantiator": "error",
        "noPrototypeBuiltins": "error",
        "noRedeclare": "error",
        "noShadowRestrictedNames": "error",
        "noUnsafeDeclarationMerging": "error",
        "noUnsafeNegation": "error",
        "useGetterReturn": "error",
        "useValidTypeof": "error"
      }
    }
  },
  "javascript": {
    "formatter": {
      "jsxQuoteStyle": "single",
      "quoteProperties": "asNeeded",
      "trailingCommas": "none",
      "semicolons": "asNeeded",
      "arrowParentheses": "always",
      "bracketSpacing": true,
      "bracketSameLine": false,
      "quoteStyle": "single",
      "attributePosition": "auto"
    }
  },
  "overrides": [
    {
      "include": ["*.ts", "*.tsx", "*.mts", "*.cts"],
      "linter": {
        "rules": {
          "correctness": {
            "noConstAssign": "off",
            "noGlobalObjectCalls": "off",
            "noInvalidConstructorSuper": "off",
            "noInvalidNewBuiltin": "off",
            "noNewSymbol": "off",
            "noSetterReturn": "off",
            "noUndeclaredVariables": "off",
            "noUnreachable": "off",
            "noUnreachableSuper": "off"
          },
          "style": {
            "noArguments": "error",
            "noVar": "error",
            "useConst": "error"
          },
          "suspicious": {
            "noDuplicateClassMembers": "off",
            "noDuplicateObjectKeys": "off",
            "noDuplicateParameters": "off",
            "noFunctionAssign": "off",
            "noImportAssign": "off",
            "noRedeclare": "off",
            "noUnsafeNegation": "off",
            "useGetterReturn": "off"
          }
        }
      }
    }
  ]
}
```

:::

マイグレーション前後でどう変わったのか差分を取っておけば良かったです…
動くか確認してみます。Biome は以下の 3 つのコマンドで実行できます。

- format : ファイルやディレクトリをフォーマットを実行
- lint: ファイルやディレクトリに対して Lint を実行
- check: format と lint の両方を実行

そしてオブションで`--write`をつけることで自動的に修正も行ってくれます。
そして`scripts`内に記述した内容が以下のようになりました。

```json
"scripts": {
  "lint": "bunx @biomejs/biome lint --write ./src",
  "format": "bunx @biomejs/biome format --write ./src",
  "check": "bunx @biomejs/biome check --write ./src"
}
```

それぞれ実行してみましたが無事動作しました！
移行時の差分が詳しく見たい方は下記の PR で確認できます。

@[card](https://github.com/maeta187/TypeScript/pull/16)

## まとめ

移行後の使用感は軽くしか触っていませんが、ESLint や Prettier とほとんど変わらないので概ね問題ないと思います。
ただ、長々とルールが羅列されると抵抗感は生まれるかもです。
Rome 時代から存在は知っていましたが、一時期プロジェクトが頓挫した時は残念に思いましたが、引き継いで使えるものになったのはありがたいです 🙏
今回は引越しも控えている中でできるだけ早く環境を新しくしたいこともあり Biome へ移行と言う方法にしましたが、ESLint の`9.x`以降のキャッチアップも行います！

## 参考

https://biomejs.dev/ja/

https://eslint.org/

https://prettier.io/
