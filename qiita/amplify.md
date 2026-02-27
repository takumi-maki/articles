# Amplify Data だけでは足りなくなったとき、どうするか ― CSV ダウンロード機能を例に考える

## 0. はじめに

Amplify Gen 2 を使うと、データの CRUD を簡単に実装できます。「まずは動くものを作りたい」場面では、Amplify Data はとても便利です。

一方で、運用していると CRUD だけでは対応しづらい要件が追加されることがあります。

この記事では、Amplify Data で構築した CRUD システムに CSV ダウンロード機能を追加するケースを例に、Amplify Data だけでは足りなくなったときの考え方を整理します。

## 1. システムの要件と技術選定

「社員管理システム、簡単でいいから作っておいて」

こうした雑な要件を前提に、「まずは動くもの」を素早く作るため、Amplify Gen 2 の Amplify Data を使ってシンプルな CRUD システムを構築するケースを考えます。

## 2. Amplify Data とは

AWS の公式ドキュメントによると、Amplify Data は以下のように説明されています。

> With Amplify Data, you can build a secure, real-time API backed by a database in minutes. After you define your data model using TypeScript, Amplify will deploy a real-time API for you. This API is powered by AWS AppSync and connected to an Amazon DynamoDB database.

([出典: AWS Amplify Documentation](https://docs.amplify.aws/vue/build-a-backend/data/set-up-data/))

Amplify Data の特徴は以下です。

- TypeScript でデータモデルを定義すると、リアルタイム API を自動生成
- AWS AppSync と Amazon DynamoDB を使用
- CRUD 操作に特化した機能

今回のように「データをそのまま管理したい」ケースでは、かなり割り切って使える仕組みだと感じました。

## 3. Amplify Data での実装内容

社員管理の CRUD は、Amplify Data で簡単に実装できました。

### スキーマ定義（amplify/data/resource.ts）

```typescript
import { type ClientSchema, a, defineData } from '@aws-amplify/backend';

const schema = a.schema({
  Employee: a
    .model({
      name: a.string().required(),
      email: a.email().required(),
      position: a.string(), // 役職
      createdAt: a.datetime(),
    })
    .authorization((allow) => [allow.publicApiKey()]),
});

export type Schema = ClientSchema<typeof schema>;

export const data = defineData({
  schema,
});
```

### フロントエンドでの使用例

スキーマを定義するだけで、CRUD 操作が使えるようになります。

```typescript
import { generateClient } from 'aws-amplify/data';
import type { Schema } from '../amplify/data/resource';

const client = generateClient<Schema>();

// 社員一覧を取得
const { data: employees } = await client.models.Employee.list();

// 社員を作成
await client.models.Employee.create({
  name: '田中太郎',
  email: 'tanaka.taro@jsl.co.jp',
  position: '主任',
});

// 社員の役職を更新
await client.models.Employee.update({
  id: '01234567-89ab-cdef-0123-456789abcdef',
  position: '課長',
});

// 社員を削除
await client.models.Employee.delete({
  id: '01234567-89ab-cdef-0123-456789abcdef',
});
```

これだけで、社員管理の基本機能が動きます。DynamoDB のテーブル作成や API の設定は、Amplify が自動でやってくれます。

## 4. 追加要件と Amplify Data の限界

CRUD システムが動き始めた後、よくある追加要件が来ました。

「社員情報を CSV でダウンロードできるようにしてほしい」

この要件に対して、Amplify Data で実装しようとすると、「どこに CSV 生成の処理を書くべきか」で少し迷いました。

Amplify Data は CRUD 中心の設計です。一方、CSV ダウンロードは「データの保存」ではなく、「成果物を生成する」処理にあたります。そのため、モデル定義の中に CSV 生成ロジックを入れるのは、責務として少しズレていると感じました。

ここで、Amplify Functions という選択肢が出てきます。

## 5. Amplify Functions とは

AWS の公式ドキュメントによると、Amplify Functions は以下のように説明されています。

> AWS Amplify Gen 2 functions are AWS Lambda functions that can be used to perform tasks and customize workflows in your Amplify app. Functions can be written in Node.js, Python, Go, or any other language supported by AWS Lambda.

([出典: AWS Amplify Documentation](https://docs.amplify.aws/vue/build-a-backend/functions/custom-functions/))

Amplify Functions の特徴は以下です。

- AWS Lambda を使用したカスタム関数
- タスクの実行やワークフローのカスタマイズが可能
- Node.js、Python、Go など、Lambda がサポートする言語で記述できる

今回のような「データを加工して成果物を作る」処理には、こちらの方が向いていそうです。

## 6. Amplify Functions での実装内容

CSV ダウンロード機能を実装するにあたって、以下のような構成にしました。

### ファイル構成

```
amplify/functions/export-csv/
├── resource.ts      # Lambda の定義
├── handler.ts       # エントリーポイント（ログ、service 呼び出し）
├── service.ts       # ビジネスロジック
├── package.json     # 依存関係の管理
└── lib/
    ├── employeeRepository.ts  # DynamoDB アクセス
    ├── csvFormatter.ts        # CSV 変換
    └── s3Storage.ts           # S3 アップロード
```

### amplify/data/resource.ts

Amplify Data のスキーマに、Lambda を統合します。

```typescript
import { type ClientSchema, a, defineData } from '@aws-amplify/backend';
import { exportCsv } from '../functions/export-csv/resource';

const schema = a.schema({
  Employee: a
    .model({
      name: a.string().required(),
      email: a.email().required(),
      position: a.string(), // 役職
      createdAt: a.datetime(),
    })
    .authorization((allow) => [allow.publicApiKey()]),

  // CSV エクスポート用の Query
  exportEmployeesCsv: a
    .query()
    .arguments({
      // 引数は不要な場合もあるが、拡張性のために残す
    })
    .returns(a.string()) // S3 の key を返す
    .authorization((allow) => [allow.publicApiKey()])
    .handler(a.handler.function(exportCsv)),
});

export type Schema = ClientSchema<typeof schema>;

export const data = defineData({
  schema,
});
```

この構成にすることで、Amplify Data の CRUD と Lambda のカスタムロジックを、同じ GraphQL API で扱えるようになります。

### amplify/functions/export-csv/resource.ts

Lambda 関数を定義します。

```typescript
import { defineFunction } from '@aws-amplify/backend';

export const exportCsv = defineFunction({
  name: 'export-csv',
});
```

### package.json

AWS SDK などの依存関係を管理します。

```json
{
  "name": "export-csv",
  "version": "1.0.0",
  "type": "module",
  "dependencies": {
    "@aws-sdk/client-dynamodb": "^3.0.0",
    "@aws-sdk/client-s3": "^3.0.0"
  }
}
```

### handler.ts

エントリーポイントはシンプルに。ログを出力して、service を呼び出すだけです。

```typescript
import { exportEmployeesToCsv } from './service';

export const handler = async () => {
  console.log('CSV export started');

  const result = await exportEmployeesToCsv();

  console.log('CSV export completed', result);
  return result;
};
```

### service.ts

ビジネスロジックはここに集約します。各処理は lib/ 配下のモジュールに分離しています。

```typescript
import { fetchAllEmployees } from './lib/employeeRepository';
import { convertToCsv } from './lib/csvFormatter';
import { uploadCsv } from './lib/s3Storage';

export const exportEmployeesToCsv = async () => {
  // 1. 社員データを取得
  const employees = await fetchAllEmployees();

  // 2. CSV に変換
  const csv = convertToCsv(employees);

  // 3. S3 にアップロード
  const s3Key = await uploadCsv(csv);

  return s3Key;
};
```

※ 各モジュール（employeeRepository, csvFormatter, s3Storage）の実装は省略していますが、DynamoDB の Scan を使う場合は、件数が増えると Query や分割処理、非同期化などを検討する必要があります。本記事では構成の説明を優先し、実装は簡略化しています。

## 7. まとめ

- Amplify Data は CRUD 中心の処理に向いている
- CSV ダウンロードのような成果物生成は Amplify Functions が適している
- 既存の Amplify Data を置き換える必要はない

まずは Amplify Data でシンプルに作り、足りなくなったところに Functions を足す、という使い方が現実的です。
