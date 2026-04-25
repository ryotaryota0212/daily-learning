# 非同期処理・メッセージキューの設計パターン

## 概要

同期処理だけで構成されたシステムは、1箇所の遅延が全体に波及する。
非同期処理を導入すると「重い処理の切り離し」「負荷の平準化」「サービス間の疎結合」が実現できる。
AI時代では推論APIの呼び出し（数秒〜数十秒）が増え、非同期設計の重要性はさらに高まっている。
設計を誤ると「処理の消失」「重複実行」「順序の逆転」など、デバッグ困難な障害を招く。

## 主要パターン

### 1. タスクキュー（Worker型）
- APIがリクエストを受け、キューにメッセージを投入 → Workerが非同期に処理
- 用途: メール送信、PDF生成、AI推論、画像処理
- Cloud Run + Cloud Tasks / Cloud Pub/Sub で実現可能

### 2. Pub/Sub（イベント駆動型）
- Publisherがイベントを発行、複数のSubscriberが独立に処理
- 用途: ユーザー登録 → 通知送信 + 分析記録 + ウェルカムメール
- サービス間の依存を切り離せる

### 3. Fan-out / Fan-in
- 1つのイベントを複数Workerに分散（Fan-out）→ 結果を集約（Fan-in）
- 用途: 大量データの並列処理、複数AI モデルへの同時推論

## べき等性（最重要概念）

メッセージは**最低1回配信（at-least-once）**が基本。つまり重複実行が前提。

- **べき等**：同じ処理を何回実行しても結果が同じ
- 実現方法：
  - リクエストに一意な `idempotency_key` を付与
  - DB側で `ON CONFLICT DO NOTHING` / `INSERT ... ON CONFLICT UPDATE`
  - 処理済みIDをテーブルに記録し、重複チェック

## アンチパターン vs 正しい設計

| アンチパターン | 問題 | 正しい設計 |
|---|---|---|
| API内で重い処理を同期実行 | タイムアウト、リソース占有 | キューに投入し即座にレスポンス |
| キュー消費の失敗を握り潰す | データ消失 | DLQ（Dead Letter Queue）に退避 |
| メッセージ順序に依存 | 順序保証なしで破綻 | べき等設計 or 順序キー指定 |
| 1メッセージ=複数副作用 | 部分失敗で不整合 | 1メッセージ=1責務に分割 |
| リトライ間隔が固定 | 障害時に負荷集中 | 指数バックオフ + ジッター |

## 設計例：Cloud Run + Pub/Sub

```python
# publisher（API側）- FastAPI
@app.post("/orders")
async def create_order(order: OrderCreate):
    order_id = str(uuid4())
    await db.execute(
        "INSERT INTO orders (id, status) VALUES ($1, 'pending')", order_id
    )
    publisher.publish(topic, json.dumps(
        {"order_id": order_id, "idempotency_key": order_id}
    ).encode())
    return {"order_id": order_id, "status": "pending"}

# subscriber（Worker側）- Cloud Run で受信
@app.post("/worker/process-order")
async def process_order(envelope: PubSubEnvelope):
    msg = decode(envelope)
    result = await db.fetchrow(
        "UPDATE orders SET status='processing' "
        "WHERE id=$1 AND status='pending' RETURNING id", msg["order_id"]
    )
    if not result:
        return {"status": "skipped"}  # べき等: 処理済みならスキップ
    await do_heavy_work(msg["order_id"])
    await db.execute(
        "UPDATE orders SET status='completed' WHERE id=$1", msg["order_id"]
    )
    return {"status": "ok"}
```

## トレードオフ

| 観点 | 同期処理 | 非同期処理 |
|---|---|---|
| 実装の単純さ | シンプル | キュー・Worker・DLQ等が必要 |
| レイテンシ | 即座に結果取得 | ポーリング or Webhook が必要 |
| スケーラビリティ | APIがボトルネック | Worker を独立スケール可能 |
| 障害の影響範囲 | 連鎖しやすい | キューがバッファとなり遮断 |
| デバッグ | スタックトレースで追跡 | 分散トレーシングが必要 |
| コスト | リクエスト比例 | キュー + Worker の常時/起動コスト |

## チェックリスト

- [ ] 全メッセージ処理がべき等になっているか（重複実行で不整合が起きないか）
- [ ] DLQ（Dead Letter Queue）を設定し、失敗メッセージの監視・アラートがあるか
- [ ] リトライは指数バックオフ + ジッターを採用しているか
- [ ] メッセージのスキーマバージョニング（後方互換性）を考慮しているか
- [ ] 処理状態をDBに記録し、クライアントがポーリングで確認できるか
