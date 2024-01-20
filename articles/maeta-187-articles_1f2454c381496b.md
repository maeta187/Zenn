---
title: 'フロントエンジニア向け!「DDL?トランザクション？なにそれ美味しいの？」を不味くする'
emoji: '🤮'
type: 'tech'
topics: [PostgreSQL, SQL, TypeScript, Prisma]
published: false
---

## はじめに

今年から転職し、新しい環境で開発をしています。
早速振られたタスクが「 DDL をトランザクションにする」というものでした。
どちらとも初めて聞くワードだったので DDL とトランザクションについて調べてみました。
最近は JS 製の ORM も多く、フロントエンドも DB 周りを開発する機会が増えてきたので自分と同じような方の参考になれば幸いです。

:::message
記事の中で SQL を使用していますが、PostgreSQL の仕様に基づいて書いています。
:::

## DDL とは

DDL（Data Definition Language）とは、データ定義言語と呼ばれる SQL の命令のことです。
DDL はデータベースのテーブルやビューを作成する**CREATE 文**や、テーブル、ビュー、インデックスなどの構造を変更する**ALTER 文**、テーブルを削除する**DROP 文**、テーブル内のデータを全削除する TRUNCATE 文などがあります。
ちなみに SELECT、INSERT、UPDATE、DELETE などのデータ操作を行う SQL 文は データ定義言語と言い、DML（Data Manipulation Language）と呼ばれます。

## トランザクション とは

データベースにおける一連の処理を 1 つのまとまりとして扱う仕組みです。
トランザクション内の処理は、全て成功するか全て失敗するかのいずれかであり、途中で中断されることはありません。これにより、データの整合性を保つことができます。
JavaScript でいうところの バラバラになっている処理を 1 つの関数にまとめ、処理を`try-catch`で処理するのに似ています。

## DDL をトランザクションにするとは

まずトランザクションされていない DDL を実行してみます。

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    stock INTEGER
    user_id INTEGER REFERENCES users(id)
);
```

この処理は`users`テーブルを作成し、その後に`products`テーブルを作成しています。
もし、`users`テーブルの作成に成功し、`products`テーブルの作成に失敗した場合、`users`テーブルは作成されたままになります。
場合によってはデータベースの状態が不整合になってしまうケースがあります。
そう言ったことを防ぐために、トランザクションを使用します。

```sql
-- トランザクションを開始する
BEGIN;

CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    stock INTEGER
    user_id INTEGER REFERENCES users(id)
);

-- ここでエラーが発生した場合の処理
ROLLBACK
-- 成功した場合の処理
COMMIT;
```

新たに`BEGIN`、`ROLLBACK`、`COMMIT`という文が追加されています。

- **BEGIN**：トランザクションを開始する
  この宣言以降の SQL 文はトランザクションに含まれる
- **ROLLBACK**：トランザクションを中断する
  トランザクションを中断し、トランザクション開始前の状態に戻す
- **COMMIT**：トランザクション内のデータベース操作を確定する
  トランザクション内のデータベース操作を確定し、データベースへ反映する

`ROLLBACK`が実行されると`COMMIT`が実行されるまでの処理は全て無効になります。
今回の場合、`ROLLBACK`が実行されると`users`テーブルも`products`テーブルも作成されません。
エラーが発生しなかった婆は`ROLLBACK`が実行されず、`COMMIT`が実行され、`users`テーブルと`products`テーブルが作成されます。
また、`SAVEPOINT チェックポイント名`で途中の処理を保存することができ、`ROLLBACK TO チェックポイント名`でそこまでロールバックします。

```sql
-- トランザクションを開始する
BEGIN;

CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);

-- ここでチェックポイントを作成する
SAVEPOINT my_savepoint;

CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    stock INTEGER
    user_id INTEGER REFERENCES users(id)
);

-- my_savepointまでロールバックする
ROLLBACK TO my_savepoint;
-- 成功した場合の処理
COMMIT;
```

上記の場合はロールバックが発生すると`users`テーブルは作成されますが、`products`テーブルは作成されません。

### 注意点

PostgreSQL は DDL をロールバックすることができますが、他のデータベースではできない場合があります。
参考の記事を貼っておきます。

https://zenn.dev/sayu/articles/58d6abfe33759f

## Prisma で DDL をトランザクションにする

### トランザクションを使用しない場合

Prisma は TypeScript 製の ORM です。
Prisma にも transaction API が提供されているのでトランザクションを使用することができます。
今回は下記の seed ファイルを使用して DB にデータを保存するようにします。

```tsx:prisma/seed.ts
import { PrismaClient } from '@prisma/client'

const prisma = new PrismaClient()

async function main() {
  try {
    const newUser = await prisma.user.create({
      data: {
        id: 1,
        name: 'John Smith'
      }
    })
    await prisma.product.create({
      data: {
        id: 1,
        name: 'Sample Product',
        stock: 10,
        userId: newUser.id
      }
    })
  } catch (e) {
    console.log(e)
  }
}

main()
  .catch((e) => {
    throw e
  })
  .finally(async () => {
    await prisma.$disconnect()
  })
```

動作確認のためまずはこのまま実行してみます。

```bash
$ pnpm ts-node --esm prisma/seed.ts
```

Prisma は DB の中を確認する GUI ツールが提供されているのでそちらで確認してみます。

```bash
$ pnpm prisma studio
```

![userテーブルの画像](/images/db-ddl-1.png)

![productテーブルの画像](/images/db-ddl-2.png)

それぞれテーブルが作れていることが確認できました！
では次は `user`の`id`を変更し、`product`の`id`を前回のままにします。
それぞれの`id` はユニークな値であるため、重複するとエラーが発生します。
今回は`product`の`id`を重複させてエラーを発生させてみます。

```tsx:prisma/seed.ts
/// ...

async function main() {
  try {
    const newUser = await prisma.user.create({
      data: {
        id: 2, // idを変更
        name: 'John Smith'
      }
    })
    // ここでエラーを発生させる
    await prisma.product.create({
      data: {
        id: 1, // 前回のまま
        name: 'Sample Product',
        stock: 10,
        userId: newUser.id
      }
    })
  } catch (e) {
    console.log(e)
  }
}
/// ...
```

`id`の完了したら再度 seed ファイルを実行し、DB を確認してみます。
下記のエラーが出れば`product`の`id`が重複しているエラーなので一旦無視で大丈夫です。

> PrismaClientKnownRequestError:
> Invalid `prisma.product.create()` invocation:
> Unique constraint failed on the fields: (`id`)

![userテーブルの画像](/images/db-ddl-3.png)

![productテーブルの画像](/images/db-ddl-4.png)

新たな`user`は作成されていますが、`product`は作成されていません。
`product`は`userId`が設定されているので`product`を`userId`で検索した時に見つからないケースが発生します。
これが不整合の例です。
そう言ったことを防ぐために、トランザクションを使用します。

### トランザクションを使用する場合

`await prisma.$transaction([])`を使用してトランザクションを使用します。
`$transaction`の引数には`Promise`を返す関数を渡します。

```tsx:prisma/seed.ts
/// ...

async function main() {
  try {
    const [user, product] = await prisma.$transaction([
      prisma.user.create({
        data: {
          id: 1,
          name: 'John Smith'
        }
      }),
      prisma.product.create({
        data: {
          id: 1,
          name: 'Sample Product',
          stock: 10,
          userId: 1
        }
      })
    ])
    console.log(user)
    console.log(product)
  } catch (e) {
    console.log(e)
  }
}

/// ...
```

先ほどまではそれぞれ変数に代入していましたが、`$transaction`を使用すると配列で返ってくるので、分割代入を使用しています。
1 度データべースを削除してから seed ファイルを実行してみます。

![userテーブルの画像](/images/db-ddl-5.png)

![productテーブルの画像](/images/db-ddl-6.png)

無事にデータが作成されていることが確認できました！
では次に先ほど同様に`user`の`id`を変更し、`product`の`id`をそのままにします。
seed ファイルを再度実行して先ほどと同様の重複エラーが発生することを確認します。
同じエラーであれば問題ないので DB を確認してみます。

![userテーブルの画像](/images/db-ddl-7.png)

![productテーブルの画像](/images/db-ddl-8.png)

`product`は作成されていないのも勿論ですが、`user`も作成されていません。
`product`作成時に重複エラーが発生したためロールバックが発生し、`user`も seed ファイル実行前に戻っています。
これがトランザクションです。

## トランザクションのデメリット

トランザクションにはデメリットもあります。
自分なりに調べましたが、DB の理解が浅いため、他にも考えられるデメリットがあるかもしれません。

- パフォーマンスの低下
  複数のトランザクションを実行すると並列処理されることがあるので、パフォーマンスが低下する可能性がある
- ロック競合
  複数のトランザクションが同時に同じリソースにアクセスしようとするときに発生する
  これにより、トランザクションの実行が遅延する可能性がある
- デッドロック
  複数のトランザクションがお互いに必要とするリソースをロックし合い、その結果、どのトランザクションも進行できなくなる状態
  トランザクション同士がお互いに必要なリソースを解放することなく待ち続けることになる

## まとめ

今回は DDL とトランザクションについて自分なりに調べた内容をまとめてみました。
タスクが振られた時は「オワター＼(^o^)／」と思っていましたが、調べてみると理解でき、タスクも無事に終わらせることができました。
サーバーサイドの開発ではこう言ったことも考えながら開発しているのだと感じ、とても勉強になりました。
新しい会社では「フルスタックになってね」と言われているので、フロントエンド、サーバサイド、インフラとしっかり勉強していきたいと思います。

### トランザム!!

## 参考

https://medium-company.com/ddl/

https://e-words.jp/w/DDL.html

https://www.prisma.io/docs/orm/prisma-client/queries/transactions

https://www.postgresql.jp/document/14/html/tutorial-transactions.html
