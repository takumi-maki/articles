# レイヤードアーキテクチャ入門：Controller・Service・Repository の責務を整理する

## はじめに

既存のプロジェクトにアサインされて、自分の最初のタスクはバグ修正でした。

コードを開いてみると、こんなフォルダ構成でした。

```
app/
├── controllers/
│   └── cart_controller.py
├── services/
│   └── cart_service.py
├── repositories/
│   └── cart_repository.py
└── models/
    └── cart_item.py
```

なんとなく意味は分かるけど、どこに何を書くべきかは分かりません。とりあえず動けばいいかと、Controllerにロジックをどんどん書きました。

レビューでこんな指摘が入りました。

![CodeRabbitのレビューコメント](ここにキャプチャのURLを入れる)

アーキテクチャ...。聞いたことはあったけど、ちゃんと理解していませんでした。

そこから構造を調べ直して、コードを直しました。

## レイヤードアーキテクチャとは

レイヤードアーキテクチャは、役割ごとにコードを層（レイヤー）に分ける構造です。それぞれの層が「自分の仕事だけ」をやります。

（図10-1：レイヤードアーキテクチャにおける標準的な論理レイヤー）

> ほとんどのレイヤードアーキテクチャは、プレゼンテーション層、ビジネス層、永続化層、データベース層の4つの標準的なレイヤーで構成されている。
>
> ——『ソフトウェアアーキテクチャの基礎』10章

## 登場人物の役割

まず、このプロジェクトで登場するクラスの役割を整理します。

| 役割        | 一言でいうと   |
| ----------- | -------------- |
| Controller  | 受付・振り分け |
| Request     | 入力チェック   |
| Service     | 業務ロジック   |
| Entity      | DBの1レコード  |
| ValueObject | ルール・計算   |
| Repository  | DBの読み書き   |

## 処理の流れ（全体像）

ユーザーの操作は、以下のように流れていきます。

1. Controller がリクエストを受け取る
2. Service に処理を依頼する
3. Entity / ValueObject で状態やルールを扱う
4. Repository を通してデータを保存する

![Geminiで生成した処理の流れの図](ここにキャプチャのURLを入れる)

ポイントは、それぞれが「隣の人の仕事を知らない」ということです。

Controllerさんは、Requestの中身をチェックして、Serviceさんに「これお願い」と渡すだけです。Serviceさんがどうやって処理しているかは知りません。

Serviceさんは、業務の流れを組み立てますが、データがどこに保存されるかは知りません。「保存して」とRepositoryさんに頼むだけです。

Repositoryさんは、DBとやりとりしますが、なぜその保存が必要なのかは知りません。言われた通りに保存するだけです。

この「知らない」が大事です。知らないからこそ、どこかを変えても他に影響が出にくくなります。疎結合というやつですね。

## カート追加の処理を追ってみる

:::note
コードは責務の理解を優先してシンプルにしています。実務では認可・トランザクション管理・バリデーションなどが加わります。
:::

### Controllerは「受け取って渡す」

最初に自分がやらかしたのがここです。Controllerに在庫チェックや価格計算を全部書いてしまいました。

Controllerの本来の役割はシンプルです。

- リクエストを受け取る
- 適切な処理に渡す

それだけです。業務ロジックは持ちません。

```python
from fastapi import APIRouter
from pydantic import BaseModel

router = APIRouter()

class AddToCartRequest(BaseModel):
    product_id: int
    quantity: int

@router.post("/cart")
def add_to_cart(req: AddToCartRequest):
    return cart_service.add(req.product_id, req.quantity)
```

### Serviceは「業務の流れを組み立てる」

Controllerから渡された処理を、Serviceが組み立てます。

- 在庫確認
- 価格取得
- カートへの追加

```python
class CartService:
    def add(self, product_id: int, quantity: int):
        product = product_repository.find(product_id)
        cart_item = CartItem(product, quantity)  # Entity を生成
        cart_repository.save(cart_item)
```

### Entityは「状態を持つ」

Entityはデータと状態の整合性を持つオブジェクトです。

```python
class CartItem:
    def __init__(self, product, quantity):
        if quantity <= 0:
            raise ValueError("数量は1以上である必要があります")
        self.product = product
        self.quantity = quantity

    def total_price(self):
        return self.product.price * self.quantity
```

「データ + ルール」まで持って初めてEntityです。

### ValueObjectは「ルールを持つ」

不変の値とそのルールを表します。

```python
class Price:
    def __init__(self, base_price: int):
        self.base_price = base_price

    def with_tax(self):
        return int(self.base_price * 1.1)
```

### Repositoryは「保存と取得をする」

Repositoryはデータアクセスを担当します。

```python
class CartRepository:
    def __init__(self, db: Session):
        self.db = db

    def save(self, cart_item: CartItem):
        self.db.add(cart_item)
        self.db.commit()
```

## なぜ分けるのか

レビューで指摘されたとき、正直「そんなに大事？」と思っていました。

でも直してみて分かりました。Controllerを開いたとき、「受け取って渡すだけ」のコードしかないので、処理の流れが追いやすい。逆にここにロジックを書き始めると、一気に見通しが悪くなります。DB構造を変えたいときはRepositoryだけ見ればいい。どこに何があるかが分かるから、修正も迷わなくなりました。

## おわりに

既存のプロジェクトにアサインされて、自分の最初のタスクはバグ修正でした。

あのとき `cart_controller.py` を開いて、どこに何を書けばいいか分からなかった自分が、CodeRabbitの指摘をきっかけに構造を調べ直して、コードを直しました。

知らないアーキテクチャのプロジェクトにアサインされて困っている人の参考になれば。
