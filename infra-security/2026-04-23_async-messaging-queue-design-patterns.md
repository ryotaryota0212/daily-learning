# 非同期処理・メッセージキューの設計パターン

## 概要

同期的なリクエスト・レスポンスだけでシステムを組むと、1箇所の遅延が全体を巻き込む。
非同期処理とメッセージキューを正しく使うことで、**コンポーネント間の結合度を下げ、障害の連鎖を防ぎ、負荷を平準化**できる。
AI時代においても「どの処理を非同期に切り出すか」「失敗時にどう復旧するか」の設計判断は人間の仕事であり、システムの信頼性を決定づける。

## 仕組みの要点

### 同期 vs 非同期の判断基準

| 同期が適切 | 非同期が適切 |
|---|---|
| ユーザーが即座に結果を必要とする | 処理に時間がかかる（メール送信、画像処理、AI推論） |
| 処理が軽量（<100ms） | 失敗時にリトライしたい |
| トランザクション整合性が必須 | 複数サービスへのファンアウトが必要 |
| 単一サービス内で完結 | ピーク負荷を平準化したい |

### 主要パターン

- **Point-to-Point（キュー）**: 1つのメッセージを1つのワーカーが処理。タスク分散に使う
- **Pub/Sub**: 1つのイベントを複数のサブスクライバーが受信。イベント駆動に使う
- **イベントソーシング**: 状態変更をイベントとして記録。監査・リプレイが必要な場合

### GCP スタックでの選択肢

- **Cloud Tasks**: Point-to-Point。Cloud Run へのHTTPタスク配信に最適
- **Cloud Pub/Sub**: Pub/Sub。サービス間のイベント連携に使う
- **Cloud Scheduler**: 定期実行。cron的なバッチ処理

## アンチパターン vs 正しい設計

| アンチパターン | 問題 | 正しい設計 |
|---|---|---|
| APIハンドラ内で重い処理を同期実行 | タイムアウト、ユーザー体験悪化 | キューに投入してすぐ202を返す |
| リトライ時に副作用が重複 | 二重課金、二重送信 | べき等性キーで重複排除 |
| 失敗メッセージを無限リトライ | キュー詰まり、リソース浪費 | Dead Letter Queue（DLQ）で隔離 |
| メッセージ順序に依存した設計 | 順序保証のコストが高い | イベントにタイムスタンプを持たせ、消費側で順序を解決 |
| キューの監視をしていない | 滞留に気づけない | キュー深度・処理遅延のアラート設定 |

## コード例：Cloud Tasks + FastAPI

```python
# タスク投入側（APIハンドラ）
@app.post("/orders", status_code=202)
async def create_order(order: OrderCreate):
    order_id = str(uuid4())
    await db.execute("INSERT INTO orders (id, status) VALUES ($1, 'pending')", order_id)
    cloud_tasks.create_task(
        queue="order-processing",
        url=f"{WORKER_URL}/tasks/process-order",
        body={"order_id": order_id, "idempotency_key": order_id},
    )
    return {"order_id": order_id, "status": "accepted"}

# ワーカー側（べき等な処理）
@app.post("/tasks/process-order")
async def process_order(task: OrderTask):
    existing = await db.fetchrow(
        "SELECT status FROM orders WHERE id = $1", task.order_id
    )
    if existing["status"] != "pending":  # べき等性チェック
        return {"status": "already_processed"}
    await execute_order_logic(task.order_id)
    await db.execute(
        "UPDATE orders SET status = 'completed' WHERE id = $1", task.order_id
    )
```

## べき等性の実装ポイント

- **DBのユニーク制約**: `idempotency_key` カラムで重複INSERTを防ぐ
- **ステータスチェック**: 処理済みなら早期リターン
- **トランザクション**: 状態更新と処理を同一トランザクションで実行

## トレードオフ

| 観点 | Cloud Tasks | Cloud Pub/Sub |
|---|---|---|
| パターン | Point-to-Point | Pub/Sub |
| ユースケース | 特定ワーカーへのタスク配信 | 複数サービスへのイベント通知 |
| 順序保証 | なし | Ordering Key指定で可能 |
| リトライ | 自動（設定可能） | Ack期限切れで再配信 |
| コスト | タスク数課金 | メッセージ量課金 |
| 適した場面 | 注文処理、メール送信 | ユーザー作成→複数サービス通知 |

## チェックリスト

- [ ] 非同期にすべき処理を特定したか（100ms超、リトライ必要、ファンアウト）
- [ ] すべてのワーカー処理にべき等性を実装したか
- [ ] Dead Letter Queue を設定し、失敗メッセージの監視・アラートがあるか
- [ ] キューの深度と処理遅延のメトリクスを監視しているか
- [ ] Cloud Run ワーカーのタイムアウトとリトライ回数を適切に設定したか
