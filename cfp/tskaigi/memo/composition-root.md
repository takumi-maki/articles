# TSKAIGIプロポーザル メモ

## タイトル
「Amplify Lambda で DDD を諦めなかった話 - Composition Root への道」

## 対象
10分LT

## ストーリーライン

### 1. 動機（1分）
「Amplify で DDD やりたかった」
- クリーンアーキテクチャ
- ドメインロジックの分離
- テスタビリティ

### 2. 問題発生（2分）
「でも Lambda は入口が1つ」

```typescript
// Handler が依存の生成場所になってしまう
export const handler = async (event) => {
  const dynamo = new DynamoDBClient({});
  const opensearch = new Client({...});
  const repo = new AsinPoolRepository(dynamo);
  const gateway = new OpenSearchGateway(opensearch);
  const domainService = new AsinDomainService(repo, gateway);
  const appService = new AsinAssignmentService(domainService, ...);
  
  return appService.execute(event);
}
```

**問題:**
- Handler が技術詳細を知りすぎ
- 依存が集中してドメインが崩れた
- 毎回クライアント生成（ウォームスタート活かせない）

### 3. 気づき（1.5分）
「依存の組み立てを外に出す必要があった」

DDD の原則:
- Handler = プレゼンテーション層
- 技術詳細を知るべきではない
- でも Lambda は入口が1つ...

**解決の方向性:**
依存の組み立てを別の場所に閉じ込める

### 4. 解決：Composition Root（3分）
「辿り着いた答え」

```typescript
// Composition Root - 依存の組み立て専用
class ProductCompositionRoot {
  private static service: Service | null = null;
  
  static async getService() {
    if (!this.service) {
      this.service = await this.create();
    }
    return this.service;  // ウォームスタート時は再利用
  }
}

// Handler はシンプルに
export const handler = async (event) => {
  const service = await ProductCompositionRoot.getService();
  return service.execute(event);
};
```

**何が変わったか:**
- Handler は技術を知らない
- 依存の組み立てが一箇所に
- ドメイン層が守られた

### 5. Lambda 特有のメリット（1.5分）
「Lambda だからこそ意味がある」

- **グローバルスコープ再利用**: 同一コンテナ内で static 変数保持
- **ウォームスタート最適化**: クライアント再生成を回避
- **テスト時差し替え**: モック注入可能

⚠️ 注意: コールドスタート改善ではなく、ウォームスタート最適化

### 6. まとめ（1分）
「Amplify Lambda でも DDD は諦めない」

- Lambda の制約を理解する
- Composition Root で依存を分離
- TypeScript だけで実現可能
- **本番環境で稼働中**

## 技術的正確性チェック

### ✅ 正確な表現
- 「ウォームスタート最適化」
- 「クライアント再生成回避」
- 「グローバルスコープ再利用」

### ❌ 避けるべき表現
- 「コールドスタート改善」（誤解を招く）
- 「パフォーマンス向上」（曖昧）

## 質疑応答想定

**Q: なぜ DI コンテナ使わない？**
A: Lambda の制約上、シンプルな実装で十分。小規模なら TypeScript だけで解決できる。

**Q: 並列実行時の安全性は？**
A: static 変数は各コンテナで独立。状態を持たないので安全。

**Q: コールドスタート改善しないの？**
A: コールドスタートは改善しません。ウォームスタート時のクライアント再利用が目的です。

**Q: 他の方法は検討した？**
A: Handler 内で全部やる、Factory パターン等も試したが、Composition Root が最もクリーンだった。

**Q: テストの並列実行は？**
A: 現状は直列実行。並列化が必要なら専用 Composition Root を検討。

**Q: 大規模になったら？**
A: その時は DI コンテナも検討。でも今は不要。

## 参考実装

- `amplify/functions/product/composition-root.ts` - Composition Root実装
- `amplify/functions/product/assign-asin/assign-asin-handler.ts` - Handler実装
- `amplify/functions/product/assign-asin/asin-assignment-application-service.ts` - Application Service

## デモ候補

1. Composition Root のコード
2. Handler のシンプルさ
3. テスト時のモック注入

## キャッチコピー候補

**推奨:**
「Amplify Lambda で DDD を諦めなかった話」

**代替:**
- 「Lambda の制約と戦った DDD 実践記」
- 「入口が1つの Lambda で Composition Root に辿り着くまで」

## ストーリーの流れ

```
理想（DDD やりたい）
  ↓
現実（Lambda は入口が1つ）
  ↓
問題（依存が集中、ドメインが崩れる）
  ↓
気づき（組み立てを外に出す）
  ↓
解決（Composition Root）
  ↓
実践（本番稼働）
```

## 削るもの

- `injectForTesting()` の詳細
- ENV 分岐のコード例
- Before/After 比較表
- DDD の理論説明
- テスト戦略の詳細

## 残すもの

- 具体的なコード（最小限）
- Lambda 特有の文脈
- 実プロジェクトで使っている事実
- ウォームスタート最適化の説明
- 問題→解決のストーリー

## 作成日
2024-12-23
