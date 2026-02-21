# GraphQLスキーマ駆動開発で実現する型安全なフルスタック開発

## TSKaigi 2025 プロポーザル案

**タイトル**: GraphQLスキーマ駆動開発で実現する型安全なフルスタック開発 - Amplify Gen2で944行のスキーマを運用した実践知見

**発表時間**: 30分

**対象者**: フルスタック開発者、型安全性を重視するエンジニア、GraphQL採用検討中のチーム

---

## 📊 現在の型定義実装状況

### 1. GraphQLスキーマ（真実の源泉）

**場所**: `amplify/data/resource.ts`

**特徴**:
- Amplify Gen2の`a.schema()`でGraphQLスキーマを定義
- カスタム型、クエリ、ミューテーションを一元管理
- 944行の大規模スキーマ（Item, CustomItem, Category, Discount, Identifier Pool等）

```typescript
// 例: Item型定義
Item: a.customType({
  id: a.string().required(),
  sku: a.string(),
  title: a.string(),
  price: a.integer(),
  // ... 他のフィールド
})

// 例: クエリ定義
getItem: a
  .query()
  .arguments({ id: a.string() })
  .returns(a.ref('Item'))
  .authorization((allow) => [allow.authenticated()])
  .handler(a.handler.function(getItem))
```

**型エクスポート**:
```typescript
export type Schema = ClientSchema<typeof schema>;
```

---

### 2. バックエンド（Lambda関数）での型使用

#### ✅ GraphQLスキーマから型を取得している

**パターン1: ハンドラー型**
```typescript
// amplify/functions/custom-item/custom-item-types.ts
import type { Schema } from 'amplify/data/resource';

export type CreateCustomItemHandler = Schema['createCustomItem']['functionHandler'];
export type UpdateCustomItemHandler = Schema['updateCustomItem']['functionHandler'];
```

**パターン2: 引数・戻り値型**
```typescript
// amplify/functions/analytics/analytics-types.ts
import type { Schema } from 'amplify/data/resource';

export type Analytics = Schema['Analytics']['type'];
```

**パターン3: カスタム型の拡張**
```typescript
// amplify/functions/item/get/get-item-types.ts
import type { Schema } from 'amplify/data/resource';

// GraphQLスキーマの型を基本として使用
export type Item = Schema['Item']['type'];

// SearchEngine固有の詳細型は別途定義（外部システム依存）
export interface ItemDocument {
  sku?: string | null;
  title?: string | null;
  // ... SearchEngine固有のフィールド
}
```

#### ⚠️ 課題: 外部システム依存の型

**問題点**: SearchEngine、Database、DataWarehouseなど外部システムのデータ構造は、GraphQLスキーマと完全には一致しない

```typescript
// amplify/functions/custom-item/custom-item-types.ts
// APIのレスポンス全体を型に起こしていない。現状必要な項目のみ定義している。
export type PriceInfo = {
  condition?: string;
  price?: number | null;
  updated_at?: string | null;
  [key: string]: unknown; // 未定義フィールドを許容
};
```

---

### 3. フロントエンドでの型使用

#### ✅ GraphQLスキーマから型を取得している

**型定義ファイル**: `src/types/amplify-schema-types.ts`

```typescript
import type { Schema } from '../../amplify/data/resource';

// 引数型
export type CreateCustomItemArguments = Schema['createCustomItem']['args'];
export type UpdateCustomItemArguments = Schema['updateCustomItem']['args'];

// 戻り値型
export type CreateCustomItemResponse = Schema['createCustomItem']['returnType'];
export type SearchCustomItemsResponse = Schema['searchCustomItems']['returnType'];

// カスタム型
export type Store = Schema['SearchCategory']['type'];
export type Discount = Schema['Discount']['type'];
export type AvailableIdentifier = Schema['AvailableIdentifier']['type'];
```

**使用例**: コンポーネントでの型使用
```typescript
// src/components/store/StoreTable.tsx
import type { Store } from '@/types/amplify-schema-types';

interface StoreTableProps {
  stores: Store[]; // GraphQLスキーマから生成された型
  onEdit: (storeId: string) => void;
  onDelete: (storeId: string) => void;
}
```

**GraphQLクライアント**: `src/lib/amplify-graphql-client.ts`
```typescript
import { serverClient } from '@/lib/amplify-server-auth';
import type { StoreSearchResponse } from '@/types/amplify-schema-types';

export async function searchStores(): ApiResponse<StoreSearchResponse | null> {
  const { data, errors } = await serverClient.queries.searchCategories({ id });
  return { data, errors }; // 型安全なレスポンス
}
```

---

### 4. 型の伝播フロー

```
┌─────────────────────────────────────────────────────────┐
│ amplify/data/resource.ts (GraphQLスキーマ)              │
│ - a.schema()でスキーマ定義                              │
│ - export type Schema = ClientSchema<typeof schema>     │
└─────────────────────┬───────────────────────────────────┘
                      │
        ┌─────────────┴─────────────┐
        │                           │
        ▼                           ▼
┌───────────────────┐      ┌────────────────────────┐
│ バックエンド      │      │ フロントエンド         │
│ (Lambda関数)      │      │ (Next.js)              │
└───────────────────┘      └────────────────────────┘
        │                           │
        ▼                           ▼
┌───────────────────┐      ┌────────────────────────┐
│ 各関数のtypes.ts  │      │ src/types/             │
│                   │      │ amplify-schema-types.ts│
│ import type {     │      │                        │
│   Schema          │      │ import type {          │
│ } from            │      │   Schema               │
│ 'amplify/data/    │      │ } from                 │
│  resource'        │      │ '../../amplify/data/   │
│                   │      │  resource'             │
└───────────────────┘      └────────────────────────┘
        │                           │
        ▼                           ▼
┌───────────────────┐      ┌────────────────────────┐
│ handler.ts        │      │ コンポーネント         │
│ service.ts        │      │ (*.tsx)                │
│                   │      │                        │
│ 型安全な実装      │      │ 型安全なUI実装         │
└───────────────────┘      └────────────────────────┘
```

---

## 🎯 GraphQLスキーマを真実の源泉とした実装の評価

### ✅ 実現できている点

1. **単一の真実の源泉**
   - GraphQLスキーマ（`amplify/data/resource.ts`）が唯一の型定義源
   - フロントエンド・バックエンド両方で同じ型を参照

2. **型の自動生成**
   - Amplify Gen2が`Schema`型を自動生成
   - `Schema['Product']['type']`のようにアクセス可能

3. **二重管理の防止**
   - フロントエンドとバックエンドで型定義を重複させない
   - スキーマ変更時、両方に自動反映

4. **型安全性の徹底**
   - コンパイル時に型チェック
   - GraphQL操作の引数・戻り値が型安全

### ⚠️ 課題・制約

1. **外部システム依存の型**
   - SearchEngine、Database、DataWarehouseのデータ構造はGraphQLスキーマと完全一致しない
   - 外部システム固有の型は別途定義が必要（`ItemDocument`等）

2. **型の粒度**
   - GraphQLスキーマは公開APIの型
   - 内部実装の詳細な型（Repository、Gateway等）は別途定義

3. **型の拡張性**
   - GraphQLスキーマにない中間型は手動定義
   - 例: `PriceInfo`型（コメント: "APIのレスポンス全体を型に起こしていない"）

---

## 📈 TSKaigi発表構成案

### タイトル
「GraphQLスキーマ駆動開発で実現する型安全なフルスタック開発 - Amplify Gen2で944行のスキーマを運用した実践知見」

### 発表構成（30分）

#### 導入（5分）
- フルスタック開発での型の二重管理問題
- 既存アプローチ（Prisma、tRPC、GraphQL Code Generator）との比較

#### 本論（15分）

**1. GraphQLスキーマを単一の真実の源泉とする設計**
- `amplify/data/resource.ts`での一元管理
- `ClientSchema<typeof schema>`による型生成

**2. バックエンド（Lambda）での型活用**
- `Schema['operation']['functionHandler']`パターン
- ハンドラー、引数、戻り値の型安全性
- 外部システム（SearchEngine、Database）との統合パターン

**3. フロントエンド（Next.js）での型活用**
- `src/types/amplify-schema-types.ts`での型再エクスポート
- コンポーネントでの型使用
- GraphQLクライアントの型安全性

**4. 型の伝播フロー**
- スキーマ変更 → 自動型生成 → 両レイヤーに反映
- コンパイル時エラー検出

#### 実践知見（5分）

**1. 遭遇した課題**
- 外部システム依存の型（SearchEngine、Database）
- 型の粒度調整（公開API vs 内部実装）
- 944行のスキーマ管理

**2. トレードオフ**
- GraphQLスキーマに全てを定義できない
- 外部システム固有の型は別途定義が必要
- 型の拡張性 vs 一元管理

**3. ベストプラクティス**
- 型定義ファイルの分離（`*-types.ts`）
- 外部システム依存の型の明示的な分離
- 型の再エクスポートパターン

#### まとめ（5分）
- 他のスタックへの応用可能性（GraphQL Code Generator等）
- 型駆動開発の未来
- 質疑応答

---

## 🎯 審査員への刺さり度評価

### 総合評価: 8/10

### 強み

1. **完全な型の伝播**
   - GraphQLスキーマ → バックエンド（Lambda） → フロントエンド（Next.js）
   - 全レイヤーで同じ型定義を使用

2. **Amplify Gen2の先進性**
   - `ClientSchema<typeof schema>`による型生成
   - `Schema['operation']['functionHandler']`パターン
   - `Schema['CustomType']['type']`パターン

3. **実践的な課題解決**
   - 944行の大規模スキーマでの実運用
   - 外部システム（SearchEngine、Database）との統合パターン

4. **型安全性の徹底**
   - ハンドラー型: `CreateCustomItemHandler`
   - 引数型: `CreateCustomItemArguments`
   - 戻り値型: `CreateCustomItemResponse`
   - カスタム型: `Store`, `Item`, `AvailableIdentifier`

### 懸念点

1. **ニッチすぎる可能性**
   - Amplify使用者は限定的（Next.js + tRPC、Prismaユーザーの方が多い）
   - 「Amplify特化」より「GraphQL型生成のベストプラクティス」として広げる必要

2. **技術的深さの不足リスク**
   - 単なる「Amplifyの機能紹介」で終わると浅い
   - 型システムの設計思想、トレードオフの議論が必要

3. **競合との差別化**
   - GraphQL Code Generatorなど既存ツールとの比較が必須
   - Amplify固有の利点を明確にする必要

---

## 🎤 最終判断

### 採択可能性: 75-80%

### 理由
1. 実装が完全にGraphQLスキーマ駆動になっている
2. 944行の大規模スキーマでの実運用実績
3. 外部システム統合の実践的な課題と解決策
4. Amplify Gen2の型生成パターンが明確

### 特に刺さる審査員層
- フルスタック開発者
- 型安全性を重視するエンジニア
- GraphQL採用を検討中のチーム
- AWS/サーバーレスアーキテクチャ実践者
- 大規模スキーマ管理に悩むチーム

### 差別化ポイント
- Amplify Gen2の`ClientSchema`パターン（他のツールにない）
- Lambda関数での`functionHandler`型パターン
- 944行の大規模スキーマでの実運用知見
- 外部システム（SearchEngine、Database）との統合パターン

---

## 📝 改善提案

### 1. タイトルを広げる
❌ 「Amplifyにおける型定義の方法」
✅ 「GraphQLスキーマ駆動開発で実現する型安全なフルスタック開発 - Amplify Gen2実践例」

### 2. 既存ツールとの比較を含める
- GraphQL Code Generator
- Prisma
- tRPC
- それぞれのトレードオフ

### 3. デモの工夫

**Before/After比較**
```typescript
// ❌ 型なし・二重管理
// frontend/types.ts
interface Item { id: string; name: string; }
// backend/types.ts  
interface Item { id: string; name: string; } // 重複！

// ✅ GraphQLスキーマ駆動
// schema.graphql（単一の真実の源泉）
type Item @model { id: ID! name: String! }
// 自動生成された型を両方で使用
```

### 4. 技術的深さの追加
- **型の伝播フロー図**: GraphQLスキーマ → Amplify生成 → Lambda → フロントエンド
- **エッジケースの処理**: Nullable型、Union型、カスタムスカラー
- **パフォーマンス考慮**: 型生成の速度、ビルド時間への影響

---

## 📚 参考実装ファイル

### GraphQLスキーマ
- `amplify/data/resource.ts` - 944行のスキーマ定義

### バックエンド型定義
- `amplify/functions/custom-item/custom-item-types.ts`
- `amplify/functions/analytics/analytics-types.ts`
- `amplify/functions/item/get/get-item-types.ts`

### フロントエンド型定義
- `src/types/amplify-schema-types.ts`
- `src/lib/amplify-graphql-client.ts`
- `src/lib/amplify-server-auth.ts`

### 使用例
- `src/components/store/StoreTable.tsx`
- `amplify/functions/custom-item/create/create-custom-item-handler.ts`
- `amplify/functions/item/assign-identifier/assign-identifier-handler.ts`

---

## 更新履歴

- 2026-02-21: 初版作成 - TSKaigi 2025プロポーザル案
