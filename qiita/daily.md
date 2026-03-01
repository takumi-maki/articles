## はじめに

日報を手で書くのをやめました。

代わりに、既に存在している行動ログをAPI経由で取得し、Notionに集約する仕組みを作りました。

取得対象は以下です。

* Slack: 自分の発言 ( 仕事、コミュニティ ) 
* Google Calendar: 当日の予定 ( 仕事、プライベート ) 
* GitHub: Publicアクティビティ(個人開発)

## アーキテクチャ

以下構成です。

* Amazon EventBridge: 平日定時実行
* AWS Lambda（Python 3.11）: 取得・整形
* 外部API: Slack / Google Calendar / GitHub
* Notion API: 保存先

![architecture.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/680905/42c66e13-a332-4bc1-aa0d-96236b48076c.png)

Lambda内部で各APIを呼び出し、Markdown文字列を生成し、Notionのblockとして分解して保存しています。

SecretsはAWS Systems Manager Parameter Storeに保存。
Lambda実行時に取得しています。

## 実装のポイント

### Slack

`search.messages` を使用。

Bot Tokenでは制限があるため、User Tokenを利用。

```python
query = f"from:<@{user_id}> after:{after} before:{before}"
result = slack.search_messages(query=query)
```

チャンネル単位でグルーピングし、時系列順に整形。

---

### Google Calendar

サービスアカウントで `events().list()` を実行。

複数カレンダーをまとめて取得し、統合ソートしています。

```python
events = service.events().list(
    calendarId=calendar_id,
    timeMin=start,
    timeMax=end,
    singleEvents=True,
    orderBy="startTime",
).execute()
```

---

### GitHub

`/users/{username}/events` を使用。

Publicイベントのみ取得可能。

```python
headers = {"Authorization": f"Bearer {token}"}
```

PushEvent と PullRequestEvent のみ抽出しています。

※ 本仕組みでは Public イベントのみを対象としています。
業務リポジトリの private 情報は取得していません。

---

### Notion

Markdownはそのまま解釈されません。
1行ずつblockに変換して保存します。

```python
notion.pages.create(
    parent={"database_id": db_id},
    properties={...},
    children=[...]
)
```

blockは100件制限があるため、分割して `append`。

## 出力データ構造

### 一覧

- 日付
- メモ（手書き1行）

![スクリーンショット 2026-02-28 22.11.51.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/680905/0ffc6398-3676-4957-9d16-5931cd7a95b2.png)


### 本文

- 実装・作業（GitHub Public）
- 時間の使い方（Calendar）
- 思考・議論（Slack）
- 学習 (Notion)

![スクリーンショット 2026-02-28 22.19.46.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/680905/1de562ab-2d43-4af8-b7ed-12549dbc6f7e.png)

本文とプロパティを分離することで、一覧性と集計性を両立しました。
## 運用コスト

* Lambda：無料枠内
* EventBridge：ほぼ無料
* Parameter Store：無料枠内

実質コストはゼロに近いです。

## 今後やりたいこと

この仕組みはまだログ収集段階なので、今後は「活用」に寄せていきたいです。


### 習慣の継続記録

* 早起き連続日数
* 学習連続日数
* 0秒思考・瞑想の実施率

チェックリストではなく、既存データから算出する形にしたいです。

---

### Slack・メモの分析

Slackや日報メモから、以下を集計。

* ポジティブワードの出現頻度
* トラブル関連ワードの増減


感情分析まではやらず、まずはキーワードベースで傾向を見るところから始めたいです。将来的にLLMを使用することを検討中。

---

### 週報・月報への展開

蓄積したデータを使って、以下を作成。

* 週報ドラフト生成
* 月次振り返りテンプレ生成

ログを貯めるだけでなく、アウトプットとして変換できる仕組みにするのが目標です。

## おわりに

日報を手で書くのをやめました。

Slack、Google Calendar、GitHubなど、すでに存在している行動ログをAPI経由で取得し、Notionに集約しています。

新しく入力を増やすのではなく、すでにあるデータをまとめる形にしました。

日報を書く仕組みではなく、日報が生成される仕組みにしています。

## 参照

https://developers.notion.com/guides/get-started/getting-started

https://docs.slack.dev/apis/

https://developers.google.com/workspace/calendar/api/guides/overview?hl=ja