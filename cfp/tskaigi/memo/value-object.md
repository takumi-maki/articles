# Value Objectで実現する型安全なドメインモデル

## 📋 目次

1. [イントロ：stringの悲劇](#1-イントロstringの悲劇)
2. [Value Objectとは？](#2-value-objectとは)
3. [実践例：ASIN Value Object](#3-実践例asin-value-object)
4. [他のValue Object実装例](#4-他のvalue-object実装例)
5. [実際の効果](#5-実際の効果)
6. [導入のコツとアンチパターン](#6-導入のコツとアンチパターン)
7. [まとめ](#7-まとめ)

---

## 1. イントロ：stringの悲劇

### 問題提起：こんなコード、見たことありませんか？

```typescript
// ❌ すべてstring型...どれがどれ？
function assignProduct(
  userId: string,      // ユーザーID？
  asin: string,        // 商品ID？
  biblioId: string,    // 書誌ID？
  reservationToken: string  // トークン？
) {
  // 引数の順番を間違えても、コンパイルエラーにならない！
  saveToDatabase(asin, userId, biblioId); // 😱
}

// 呼び出し側も地獄
assignProduct(
  "B000000001",  // これASINじゃない？
  "user-123",    // これユーザーID？
  "doc-456",     // これ何？
  "uuid-789"     // これも何？
);
```

### 起こりうる問題

- 引数の順番を間違える
- バリデーションが散らばる
- ログ出力時に機密情報が漏れる
- テストが書きにくい

---

## 2. Value Objectとは？

### 定義

> Value Objectは、ドメイン駆動設計（DDD）における概念で、
> 「値そのもの」を表現するオブジェクト。
> 識別子ではなく、値の内容で等価性を判断する。

### 特徴

1. **不変性（Immutable）**: 一度作ったら変更できない
2. **値による等価性**: 中身が同じなら同じオブジェクト
3. **自己検証**: コンストラクタでバリデーション
4. **ドメイン知識の集約**: その値に関するロジックを一箇所に

### TypeScriptでの実装パターン

```typescript
class Asin {
  private readonly value: string;  // privateで外部から変更不可
  
  constructor(value: string) {
    // コンストラクタでバリデーション
    if (!this.isValidAsin(value)) {
      throw new Error(`Invalid ASIN: ${value}`);
    }
    this.value = value.toUpperCase();
  }
  
  // 値の取得
  getValue(): string {
    return this.value;
  }
  
  // 値による等価性
  equals(other: Asin): boolean {
    return this.value === other.value;
  }
}
```

---

## 3. 実践例：ASIN Value Object

### 背景

- ASIN = Amazon Standard Identification Number
- 形式：B + 英数字9文字（例：B000000001）
- 商品管理の中核となる識別子

### 3-1. バリデーションの集約

```typescript
class Asin {
  private readonly value: string;

  constructor(value: string) {
    if (!this.isValidAsin(value)) {
      throw new Error(
        `Invalid ASIN: ${value}. Must be B + 9 alphanumeric chars`
      );
    }
    this.value = value.toUpperCase(); // 正規化
  }

  private isValidAsin(value: string): boolean {
    if (typeof value !== 'string' || value.length !== 10) {
      return false;
    }
    // B始まり + 英数字9文字
    return /^B[A-Z0-9]{9}$/i.test(value);
  }
}
```

**メリット：**
- バリデーションロジックが一箇所に集約
- ASINを使う全ての場所で自動的にバリデーション
- 不正な値を持つASINオブジェクトは存在しない


### 3-2. 用途別メソッドで意図を明確化

```typescript
class Asin {
  // ... 省略 ...
  
  // OpenSearch検索用
  toSearchKeyword(): string {
    return this.value;
  }
  
  // S3パス生成用
  toS3SafeString(): string {
    return this.value;
  }
  
  // ログ出力用（機密情報をマスク）
  toLogSafeString(): string {
    return `${this.value.substring(0, 2)}***${this.value.substring(8)}`;
    // 例: B000000001 → B0***01
  }
  
  // URL用
  toUrlSafeString(): string {
    return encodeURIComponent(this.value);
  }
}
```

**メリット：**
- 用途が明確になる
- セキュリティ要件（ログマスキング）を一箇所で管理
- 使う側は意図を明示できる

**使用例：**

```typescript
const asin = new Asin('B000000001');

logger.info('ASIN assigned', { 
  asin: asin.toLogSafeString()  // B0***01
});

const s3Path = `products/${asin.toS3SafeString()}/image.jpg`;
```

### 3-3. 型安全性の向上

**Before（プリミティブ型）:**

```typescript
// ❌ すべてstring - 間違いに気づけない
function assignProduct(
  userId: string,
  asin: string,
  biblioId: string
) {
  // 引数の順番を間違えてもコンパイルエラーにならない
  saveToDatabase(asin, userId, biblioId); // 😱
}

assignProduct(
  "B000000001",  // これASIN？
  "user-123",    // これユーザーID？
  "doc-456"      // これ何？
);
```

**After（Value Object）:**

```typescript
// ✅ 型が違う - 間違いに気づける
function assignProduct(
  userId: UserId,
  asin: Asin,
  biblioId: BiblioId
) {
  // 型が違うのでコンパイルエラー！
  saveToDatabase(asin, userId, biblioId); // ✅ 型チェックで検出
}

assignProduct(
  new UserId("user-123"),
  new Asin("B000000001"),
  new BiblioId("doc-456")
);

// 引数の順番を間違えるとコンパイルエラー
assignProduct(
  new Asin("B000000001"),  // ❌ Type 'Asin' is not assignable to type 'UserId'
  new UserId("user-123"),
  new BiblioId("doc-456")
);
```

---

## 4. 他のValue Object実装例

### 4-1. UserId - セキュリティ重視

```typescript
class UserId {
  private readonly value: string;
  
  constructor(value: string) {
    if (!this.isValidUserId(value)) {
      throw new Error('Invalid UserId');
    }
    this.value = value;
  }
  
  // ログ出力用（マスキング）
  toLogSafeString(): string {
    if (this.value.length <= 6) return '***';
    return `${this.value.substring(0, 3)}***${this.value.substring(this.value.length - 3)}`;
  }
  
  // 監査ログ用（完全な値）
  toAuditString(): string {
    return this.value;
  }
  
  // システムアカウント判定
  isSystemAccount(): boolean {
    return ['system-', 'admin-', 'service-']
      .some(prefix => this.value.startsWith(prefix));
  }
}
```

### 4-2. ReservationToken - UUID生成

```typescript
class ReservationToken {
  private readonly value: string;
  
  // 新規生成
  static generate(): ReservationToken {
    return new ReservationToken(randomUUID());
  }
  
  // 既存値から復元
  static fromString(value: string): ReservationToken {
    return new ReservationToken(value);
  }
  
  // 期限切れ判定
  isExpired(createdAt: Date, expirationMinutes: number = 72 * 60): boolean {
    const ageMinutes = this.getAgeInMinutes(createdAt);
    return ageMinutes > expirationMinutes;
  }
}
```


### 4-3. UserGroups - コレクション型Value Object

```typescript
class UserGroups {
  private readonly groups: readonly string[];
  
  constructor(groups: string[]) {
    // 重複除去、ソート、正規化
    this.groups = [...new Set(groups.map(g => g.trim()))].sort();
  }
  
  // 権限チェック
  hasInventorySelectionAccess(): boolean {
    return this.containsAny(['bandb', 'partner', 'premium']);
  }
  
  // 集合演算
  union(other: UserGroups): UserGroups {
    return new UserGroups([...this.groups, ...other.groups]);
  }
  
  intersection(other: UserGroups): UserGroups {
    const common = this.groups.filter(g => other.contains(g));
    return new UserGroups(common);
  }
}
```

---

## 5. 実際の効果

### 5-1. バグの早期発見

**実例：引数の順番ミス**

```typescript
// Before: 実行時エラー（本番で発覚）
function confirmAsin(userId: string, asin: string, token: string) {
  // 間違った順番で呼び出されても気づけない
}
confirmAsin("B000000001", "user-123", "token-456"); // 😱

// After: コンパイルエラー（開発時に発覚）
function confirmAsin(userId: UserId, asin: Asin, token: ReservationToken) {
  // ...
}
confirmAsin(
  new Asin("B000000001"),  // ❌ コンパイルエラー！
  new UserId("user-123"),
  new ReservationToken("token-456")
);
```

### 5-2. テストの簡潔化

```typescript
// Value Objectのテスト
describe('Asin', () => {
  it('不正な形式を拒否する', () => {
    expect(() => new Asin('invalid')).toThrow();
    expect(() => new Asin('123456789')).toThrow(); // 数字のみ
    expect(() => new Asin('A000000001')).toThrow(); // A始まり
  });
  
  it('ログ出力時にマスキングされる', () => {
    const asin = new Asin('B000000001');
    expect(asin.toLogSafeString()).toBe('B0***01');
  });
});
```

### 5-3. リファクタリングの安全性

```typescript
// ASINの形式変更が必要になった場合
class Asin {
  private isValidAsin(value: string): boolean {
    // ここを変更するだけで、全ての使用箇所に反映される
    return /^B[A-Z0-9]{9}$/i.test(value);
  }
}
```

---

## 6. 導入のコツとアンチパターン

### 6-1. 段階的導入

```typescript
// Step 1: 新規機能から導入
class NewFeatureService {
  async execute(asin: Asin) {  // ✅ Value Object
    // ...
  }
}

// Step 2: 既存コードとの境界で変換
class LegacyAdapter {
  async callLegacy(asin: Asin) {
    const asinString = asin.getValue();  // string に変換
    await legacyFunction(asinString);
  }
}
```

### 6-2. 避けるべきアンチパターン

**❌ 過度な Value Object 化**

```typescript
// やりすぎ例
class ProductName {
  constructor(private value: string) {}
}
class ProductPrice {
  constructor(private value: number) {}
}
class ProductDescription {
  constructor(private value: string) {}
}
// → 単純な文字列や数値まで Value Object にする必要はない
```

**✅ 適切な判断基準**

Value Object にすべきもの：
- ドメイン固有の識別子（ASIN、UserId）
- 複雑なバリデーションルールがある
- 用途別の変換ロジックがある
- 型安全性が重要

プリミティブ型のままでよいもの：
- 単純な文字列（名前、説明）
- 単純な数値（価格、数量）
- ビジネスルールが少ない

---

## 7. まとめ

### Value Object のメリット

1. **型安全性の向上**
   - コンパイル時に間違いを検出
   - IDEの補完が効く

2. **バリデーションの集約**
   - 不正な値を持つオブジェクトは存在しない
   - テストが書きやすい

3. **ドメイン知識の可視化**
   - コードがドメインを表現する
   - 用途別メソッドで意図が明確

4. **保守性の向上**
   - 変更が一箇所で済む
   - リファクタリングが安全


### 今日から始められること

```typescript
// 1. 識別子をValue Objectに
type UserId = string;  // ❌
class UserId { ... }   // ✅

// 2. バリデーションをコンストラクタに
function validateAsin(asin: string) { ... }  // ❌
class Asin { constructor(value: string) { ... } }  // ✅

// 3. 用途別メソッドを追加
asin.substring(0, 2)  // ❌
asin.toLogSafeString()  // ✅
```

---

## 補足資料

### 実装テンプレート

```typescript
class YourValueObject {
  private readonly value: T;  // プリミティブ型
  
  constructor(value: T) {
    // 1. バリデーション
    if (!this.isValid(value)) {
      throw new Error(`Invalid ${this.constructor.name}: ${value}`);
    }
    // 2. 正規化
    this.value = this.normalize(value);
  }
  
  // 3. ファクトリーメソッド
  static fromString(value: string): YourValueObject {
    return new YourValueObject(value);
  }
  
  // 4. 値の取得
  getValue(): T {
    return this.value;
  }
  
  // 5. 等価性判定
  equals(other: YourValueObject): boolean {
    return this.value === other.value;
  }
  
  // 6. 用途別メソッド
  toDisplayString(): string { ... }
  toLogSafeString(): string { ... }
  
  // 7. バリデーション（private）
  private isValid(value: T): boolean { ... }
  
  // 8. 正規化（private）
  private normalize(value: T): T { ... }
}
```

### プロジェクトでの実装例

このトークで紹介したValue Objectは、実際のプロダクションコードから抽出しています。

**実装ファイル：**
- `amplify/functions/product/value-objects/asin.ts`
- `amplify/functions/product/value-objects/user-id.ts`
- `amplify/functions/product/value-objects/biblio-id.ts`
- `amplify/functions/product/value-objects/reservation-token.ts`
- `amplify/functions/product/value-objects/user-groups.ts`

**使用例：**
- ASIN自動割当機能での活用
- ユーザー権限管理での活用
- S3ファイルパス生成での活用

### 参考リンク

**ドメイン駆動設計（DDD）:**
- Eric Evans『Domain-Driven Design』
- Vaughn Vernon『Implementing Domain-Driven Design』

**TypeScriptでのDDD実装:**
- [TypeScript DDD Example](https://github.com/stemmlerjs/ddd-forum)
- [Value Objects in TypeScript](https://khalilstemmler.com/articles/typescript-value-object/)

**プロジェクト内ドキュメント:**
- `.kiro/steering/domain-driven-design.md` - DDDガイドライン
- `documents/guidelines/typescript.md` - TypeScript開発ガイドライン

---

## トーク構成メモ

### タイムライン（30分想定）

- **0-3分**: イントロ - stringの悲劇
- **3-8分**: Value Objectとは？
- **8-18分**: 実践例 - ASIN Value Object
- **18-23分**: 他の実装例
- **23-26分**: 実際の効果
- **26-28分**: 導入のコツ
- **28-30分**: まとめ

### デモ候補

1. **TypeScript Playground でのライブデモ**
   - プリミティブ型での型エラーが出ない例
   - Value Object での型エラーが出る例

2. **実際のプロジェクトコード**
   - Asin クラスの実装
   - 使用箇所での型安全性

3. **テストコード**
   - バリデーションのテスト
   - 用途別メソッドのテスト

### Q&A想定質問

**Q1: パフォーマンスへの影響は？**
A: オブジェクト生成のオーバーヘッドはありますが、実用上問題になることは稀です。必要に応じてファクトリーでキャッシュする方法もあります。

**Q2: 既存コードへの導入方法は？**
A: 新規機能から段階的に導入し、境界でプリミティブ型との変換を行います。一気に全体を書き換える必要はありません。

**Q3: すべての文字列をValue Objectにすべき？**
A: いいえ。ドメイン固有の識別子や複雑なバリデーションが必要なものに限定すべきです。

**Q4: Zodなどのバリデーションライブラリとの使い分けは？**
A: Zodは入力バリデーション、Value Objectはドメインモデルの表現に使います。併用も可能です。

**Q5: フロントエンドでも使える？**
A: はい。ただし、APIレスポンスからの復元処理が必要です。`fromJSON`メソッドを用意すると便利です。

---

## 作成者メモ

このトーク資料は、Valuebooks Proプロジェクトでの実装経験をもとに作成しました。

**プロジェクト背景：**
- AWS Amplify Gen2 + TypeScript
- ドメイン駆動設計の採用
- Lambda関数でのValue Object活用

**学んだこと：**
- Value Objectによる型安全性の向上
- バリデーションの一元管理
- 用途別メソッドによる意図の明確化
- テスタビリティの向上

**今後の展開：**
- Aggregate Rootパターンの活用
- Repository パターンとの組み合わせ
- イベントソーシングへの応用

---

## ライセンス

このトーク資料は、TSKAIGIでの発表用に作成されました。
プロジェクトのコード例は、実際のプロダクションコードから抽出していますが、
機密情報は含まれていません。

