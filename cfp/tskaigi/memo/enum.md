# TSKaigi プロポーザル：Amplify Gen2のenumに`.required()`がない理由

## タイトル案

**「なぜAmplify Gen2のenumに`.required()`がないのか - 型安全性とAPI設計の狭間で」**

または

**「TypeScriptの型システムから見るAmplify Gen2のenum設計思想」**

## セッション構成（10分）

### 1. 導入：問題提起（1分）

```typescript
// これは動く
status: a.string().required()

// これは動かない！
productType: a.enum(['normal', 'alcohol', 'event']).required()
//                                                   ^^^^^^^^
// Property 'required' does not exist on type 'EnumType'
```

「なぜ？」という疑問から始める

---

### 2. 型定義を追う（2分）

**ModelFieldとEnumTypeの違い**

```typescript
// ModelField - requiredメソッドがある
export type ModelField<T> = {
  required(): ModelField<Required<T>>
  array(): ModelField<ArrayField<T>>
  // ...
}

// EnumType - requiredメソッドがない
export interface EnumType<values> {
  type: 'enum'
  values: values
  // required()がない！
}
```

**なぜ分離されているのか？**

---

### 3. GraphQLの制約（2分）

**GraphQLのenum仕様**

```graphql
enum ProductType {
  NORMAL
  ALCOHOL
  EVENT
}

type Product {
  # enumは値の制約のみ
  productType: ProductType
  
  # 必須性は別の概念
  productType: ProductType!  # 必須
  productType: ProductType   # オプショナル
}
```

**Amplify Gen2の設計判断**
- GraphQLスキーマとの1:1マッピングを優先
- enumは「値の制約」、requiredは「存在の制約」
- 2つの関心事を分離

---

### 4. 実装上の理由（2分）

**型システムの複雑さ**

```typescript
// もしrequired()があったら...
type EnumField<T> = {
  required(): EnumField<Required<T>>  // ❌ Enumに対してRequiredは意味をなさない
}

// Enumの型は文字列リテラルのユニオン
type ProductType = 'normal' | 'alcohol' | 'event'

// Required<ProductType>は...？
type Result = Required<'normal' | 'alcohol' | 'event'>
// = 'normal' | 'alcohol' | 'event'  // 変わらない！
```

**TypeScriptの型レベルでの制約**
- `Required<T>`はオブジェクトのプロパティに作用
- プリミティブ型（文字列リテラル）には効果がない
- enumは既に「この値のいずれか」という制約

---

### 5. 実践的な解決策（2分）

**パターン1: デフォルト値で回避**

```typescript
// a.model()の場合
status: a.enum(['AVAILABLE', 'RESERVED', 'USED'])
  .default('AVAILABLE')  // ✅ 実質必須
```

**パターン2: バリデーション層で保証**

```typescript
// Lambda handler
if (!input.productType) {
  throw new Error('productType is required')
}

// Zodスキーマ（フロントエンド）
productType: z.enum(['normal', 'alcohol', 'event'], {
  required_error: 'Product type is required'
})
```

**パターン3: 型ガードで安全性確保**

```typescript
function isValidProductType(
  value: unknown
): value is ProductType {
  return ['normal', 'alcohol', 'event'].includes(value as string)
}
```

---

### 6. ベストプラクティス：多層防御戦略（2分）

#### 層ごとの責務分離

```typescript
// 【GraphQLスキーマ層】値の制約のみ
// amplify/data/resource.ts
const schema = a.schema({
  Product: a.model({
    // enumは「この値のいずれか」という制約
    productType: a.enum(['normal', 'alcohol', 'event']),
    // 必須性は各層で保証
  })
})
```

```typescript
// 【フロントエンド：Zodスキーマ】必須 + 型安全
// src/lib/validation/product-schema.ts
import { z } from 'zod'

export const productFormSchema = z.object({
  productType: z.enum(['normal', 'alcohol', 'event'], {
    required_error: 'Product type is required',
    invalid_type_error: 'Invalid product type'
  }),
  // ✅ ユーザー入力時点で必須チェック
  // ✅ 型安全性も保証
})
```

```typescript
// 【バックエンド：Zodスキーマ】必須 + ビジネスルール
// amplify/functions/product/create/validation.ts
import { z } from 'zod'

export const createProductInputSchema = z.object({
  productType: z.enum(['normal', 'alcohol', 'event']),
  // ✅ API境界で必須チェック
  // ✅ 不正なリクエストを早期に弾く
}).refine(
  (data) => {
    // ビジネスルール：alcoholは特定条件でのみ許可
    if (data.productType === 'alcohol') {
      return hasAlcoholLicense(data.userId)
    }
    return true
  },
  { message: 'Alcohol products require license' }
)
```

#### なぜこの設計が優れているのか

**1. 関心の分離**
```
GraphQL層    → 値の制約（enumの定義）
Validation層 → 存在の制約（required）+ ビジネスルール
Type層       → 開発時の安全性（TypeScript）
```

**2. 防御の多層化**
```
User Input → Frontend Zod → GraphQL → Backend Zod → DB
           ✅ 必須チェック  ✅ 型制約  ✅ 必須チェック
                                      ✅ ビジネスルール
```

**3. 柔軟性の確保**

```typescript
// 例：既存データとの互換性
// GraphQL: オプショナル（既存データがnullの可能性）
productType: a.enum(['normal', 'alcohol', 'event'])

// Validation: 新規作成時のみ必須
export const createSchema = z.object({
  productType: z.enum([...])  // 必須（デフォルト）
})

export const updateSchema = z.object({
  productType: z.enum([...]).optional()  // オプショナル
})
```

#### 実装例：完全版

```typescript
// 【1. GraphQL Schema】
// amplify/data/resource.ts
const createProductArgs = {
  productType: a.enum(['normal', 'alcohol', 'event']),
  // 値の制約のみ
}

// 【2. Frontend Validation】
// src/components/product/ProductEditor.tsx
import { zodResolver } from '@hookform/resolvers/zod'

const form = useForm({
  resolver: zodResolver(productFormSchema),
  defaultValues: {
    productType: 'normal'  // デフォルト値で実質必須化
  }
})

// 【3. Backend Validation】
// amplify/functions/product/create/handler.ts
export const handler = async (event: AppSyncEvent) => {
  // 入力検証
  const input = createProductInputSchema.parse(event.arguments)
  //                                     ^^^^^ 
  // ここでrequiredチェック + ビジネスルール検証
  
  // ビジネスロジック実行
  return await createProduct(input)
}
```

---

### 7. 設計思想の学び（1分）

**Amplify Gen2が教えてくれること**

1. **関心の分離**
   - 値の制約（enum）
   - 存在の制約（required）
   - 構造の制約（型システム）

2. **GraphQL First**
   - TypeScriptはGraphQLスキーマの表現手段
   - GraphQLの制約に従う設計

3. **多層防御**
   - スキーマレベル：値の制約
   - バリデーション層：ビジネスルール
   - 型レベル：開発時の安全性

---

## スライド構成案

### スライド1: タイトル
- タイトル
- 自己紹介（プロジェクト経験）

### スライド2: 問題提起
- コード例（動くもの vs 動かないもの）
- 「なぜ？」

### スライド3-4: 型定義を追う
- ModelFieldの定義
- EnumTypeの定義
- 比較表

### スライド5-6: GraphQLの制約
- GraphQL enum仕様
- Amplify Gen2の設計判断
- アーキテクチャ図

### スライド7-8: 実装上の理由
- TypeScriptの型システム
- Required<T>の動作
- コード例

### スライド9-10: ベストプラクティス
- 多層防御戦略
- 層ごとの責務分離
- 実装例

### スライド11: 設計思想の学び
- 3つのポイント
- まとめ

### スライド12: まとめ
- キーメッセージ
- 参考リンク

---

## デモコード案

**ライブコーディング（2分程度）**

```typescript
// 1. 問題の再現
const schema = a.schema({
  Product: a.model({
    name: a.string().required(),  // ✅
    productType: a.enum(['normal', 'alcohol', 'event']).required()  // ❌
  })
})

// 2. 型定義を確認
type EnumType = typeof a.enum
// => requiredメソッドがない

// 3. 解決策の実装
const schema = a.schema({
  Product: a.model({
    name: a.string().required(),
    productType: a.enum(['normal', 'alcohol', 'event'])
      .default('normal')  // ✅ 実質必須
  })
})

// 4. Zodでの多層防御
const productSchema = z.object({
  productType: z.enum(['normal', 'alcohol', 'event'])  // ✅ 必須
})
```

---

## プロポーザル文案

**タイトル**
「なぜAmplify Gen2のenumに`.required()`がないのか - 型安全性とAPI設計の狭間で」

**概要（200-300字）**
AWS Amplify Gen2でスキーマを定義する際、`a.string().required()`は動くのに`a.enum([...]).required()`は型エラーになります。この一見不可解な挙動の裏には、GraphQLの設計思想、TypeScriptの型システムの制約、そしてAPI設計における関心の分離という深い理由があります。本セッションでは、型定義を追いながらこの設計判断の背景を解き明かし、GraphQL層・フロントエンド・バックエンドでZodを使った多層防御戦略という実践的な解決策を提示します。フレームワークの制約から学ぶ、型安全なAPI設計の考え方をお伝えします。

**対象者**
- TypeScriptで型安全なAPIを設計したい方
- AWS Amplifyを使っている/検討している方
- フレームワークの設計思想に興味がある方
- Zodを使ったバリデーション戦略に興味がある方

**得られる知見**
- GraphQLとTypeScriptの型システムの関係
- enumとrequiredの関心の分離
- Zodを使った多層防御のバリデーション戦略
- 実践的なフルスタック型安全設計

---

## 補足資料

**参考リンク集**
- [Amplify Gen2 公式ドキュメント - Data Modeling](https://docs.amplify.aws/javascript/build-a-backend/data/data-modeling/)
- [GraphQL Enum仕様](https://graphql.org/learn/schema/#enumeration-types)
- [TypeScript Utility Types](https://www.typescriptlang.org/docs/handbook/utility-types.html)
- [Zod Documentation](https://zod.dev/)
- 実装例（GitHubリポジトリ）

**QA想定**
- Q: なぜGraphQLに従う必要があるのか？
  - A: Amplify Gen2はGraphQL APIを生成するため、GraphQLスキーマとの整合性が最優先
  
- Q: 他のフレームワーク（Prisma、tRPCなど）はどうしているのか？
  - A: Prismaはenum + requiredをサポート（GraphQL非依存）、tRPCはZodスキーマベース
  
- Q: 将来的に`.required()`が追加される可能性は？
  - A: GraphQLスキーマとの1:1マッピングを維持する限り、追加は難しい。多層防御が推奨アプローチ

---

## キーメッセージ

**「制約は設計思想の表れ。フレームワークの制約を理解することで、より良いアーキテクチャが見えてくる」**

- enumに`.required()`がないのは欠陥ではなく、設計思想
- GraphQL層は値の制約、Validation層は存在の制約とビジネスルール
- Zodによる多層防御で、型安全性と柔軟性を両立

---

## 作成日
2025-01-14
