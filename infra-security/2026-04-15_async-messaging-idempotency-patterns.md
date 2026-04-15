# 非同期処理・メッセージキューの設計パターン（Pub/Sub・イベント駆動・べき等性）

## 概要
- 同期APIですべてを処理すると、外部連携・重い処理がレイテンシとSLOを壊す。非同期化は「壊れにくさ」を生む基本技法。
- ただしキューを入れた瞬間、「順序」「重複」「失敗時再送」といった分散システムの難しさが顔を出す。
- AI時代に評価されるのは「キューを入れた後に起きる事故を全部想定できる設計者」。鍵は **At-least-once前提のべき等設計**。

## 仕組みの要点
- **Producer → Broker → Consumer**: ProducerはBrokerに送ったら即返す。Consumerは別プロセスで処理。
- **配信保証の3種**:
  - At-most-once: 失えば消える（ログ向き）
  - At-least-once: 最低1回届く＝重複あり（**実務の標準**）
  - Exactly-once: 理想だが実現コスト高い
- **順序保証**: グローバル順序は高くつく。パーティションキー（例: `user_id`）単位での順序で十分なことが多い。
- **DLQ (Dead Letter Queue)**: N回失敗したらDLQへ退避。本処理を詰まらせない。
- **Outboxパターン**: DB更新とメッセージ送信を同じトランザクションで書き、別ワーカーが送る。二重書き込み問題を解く定石。

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| API内で外部API呼び出し・重い処理を同期実行 | Pub/Subに投げて即200返却、ワーカーで処理 |
| `INSERT`後に`publish()`を普通に呼ぶ | Outbox経由でトランザクション一貫性を確保 |
| Consumerでべき等性を考えない | `message_id`で処理済みチェック |
| 失敗したら無限リトライ | 指数バックオフ + 最大N回 → DLQ |
| DLQを作って放置 | DLQにアラート＋再投入フロー |

## コード例（FastAPI + Cloud Pub/Sub、べき等Consumer）
```python
# producer: Outbox経由（Neonで同一Tx）
async def create_order(db, payload):
    async with db.transaction():
        order_id = await db.fetchval("INSERT INTO orders ... RETURNING id", ...)
        await db.execute(
            "INSERT INTO outbox(aggregate_id, event_type, payload) VALUES($1,$2,$3)",
            order_id, "order.created", json.dumps(payload),
        )
    return order_id

# consumer: At-least-once前提でべき等化
async def handle(msg):
    event_id = msg.attributes["event_id"]
    async with db.transaction():
        inserted = await db.fetchval(
            "INSERT INTO processed_events(event_id) VALUES($1) "
            "ON CONFLICT DO NOTHING RETURNING event_id", event_id)
        if inserted is None:
            return msg.ack()  # 既に処理済み → ack
        await do_business_logic(json.loads(msg.data))
    msg.ack()
```

## トレードオフ

| 観点 | 同期処理 | 非同期 + キュー |
|---|---|---|
| レイテンシ（ユーザー体感） | 処理時間に比例 | 即応答（裏で処理） |
| 実装の単純さ | シンプル | Producer/Consumer/DLQ/監視が必要 |
| 失敗時の挙動 | ユーザーにエラー | 自動リトライ、運用で吸収 |
| 一貫性 | トランザクションで自然 | Outbox等の工夫が必要 |
| 観測性 | スタックトレース1本 | trace IDで串刺しが必須 |

- **判断軸**: 「3秒以上かかる or 外部依存あり or 負荷スパイクあり」なら非同期化の候補。それ以外は同期で十分。

## チェックリスト
- [ ] Consumerは同じメッセージが2回来ても安全か（`event_id` + `processed_events`テーブル等）
- [ ] DB更新とpublishは Outbox で一貫しているか
- [ ] リトライは指数バックオフ + 最大回数 + DLQ が設定されているか
- [ ] DLQの深さにアラートが付いているか、再投入手順があるか
- [ ] trace_id / correlation_id がメッセージ属性に乗り、ログで追えるか
