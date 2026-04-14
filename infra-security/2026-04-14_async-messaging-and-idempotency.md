# 非同期処理・メッセージキューの設計パターン（Pub/Sub・イベント駆動・べき等性）

## 概要
- ユーザー操作と副作用（メール送信、画像処理、集計）を切り離すと、レイテンシと可用性が大きく改善する。
- ただし「非同期化すれば速い」で済む話ではない。**at-least-once 配信**・**順序性なし**・**再送**を前提に設計しないと、二重課金や整合性崩壊を引き起こす。
- AI時代に重要なのは「どのワークロードを同期に残し、どれを非同期に逃がすか」の切り分けと、**べき等性（idempotency）** を設計の前提に組み込む力。

## 仕組みの要点
- **メッセージキュー**: 送信側（Producer）と受信側（Consumer）を疎結合にする中間層。Cloud Pub/Sub, SQS, Kafka など。
- **配信セマンティクス**:
  - at-most-once: 欠落許容（ログ送信など）
  - **at-least-once**: 現実のデフォルト。再送あり → **べき等性必須**
  - exactly-once: 実質は「べき等 + at-least-once」で達成するもの
- **ack/nack**: Consumer が処理完了を通知するまで再配信される。処理中にクラッシュすれば再配信される前提。
- **DLQ (Dead Letter Queue)**: N 回リトライ後に失敗したメッセージを隔離。データ破損の連鎖を防ぐ。
- **イベント駆動**: 「状態変更」を publish し、他サービスが subscribe する。疎結合だが追跡が難しくなる。

## アンチパターン vs 正しい設計
| アンチパターン | 正しい設計 |
|---|---|
| HTTP API の中で `send_email()` を同期呼び出し | キューに enqueue のみ。送信は worker 側 |
| DB 書き込み後に Pub/Sub publish（片方失敗で不整合） | **Transactional Outbox** パターン |
| Consumer が「初めての受信」前提で処理 | `message_id` を保存してべき等にする |
| リトライ無限・DLQ なし | 最大リトライ回数 + DLQ + アラート |
| 1 メッセージ = 1 レコードで重い処理を同期実行 | バッチ化 or fan-out |
| 順序依存のロジックをキュー前提で書く | 順序が必要ならパーティションキーで同一キーを同一 worker に寄せる |

## 設計例（FastAPI + Cloud Pub/Sub + Neon）

```python
# Producer: 注文確定 → outbox に書いて別プロセスで publish
async def create_order(db, user_id, items):
    async with db.transaction():
        order = await db.fetch_one("INSERT INTO orders ... RETURNING id", ...)
        await db.execute(
            "INSERT INTO outbox(event_type, payload) VALUES($1, $2)",
            "order.created", json.dumps({"order_id": order["id"]}),
        )
    return order  # publish は outbox relay が非同期で実行
```

```python
# Consumer: message_id でべき等化
async def handle(message):
    mid = message.message_id
    try:
        await db.execute(
            "INSERT INTO processed_messages(id) VALUES($1)", mid
        )  # UNIQUE 制約で二重処理を弾く
    except UniqueViolation:
        message.ack(); return
    await send_email(json.loads(message.data))
    message.ack()
```

## トレードオフ
| 観点 | 同期処理 | 非同期 + キュー |
|---|---|---|
| レイテンシ（ユーザー体感） | 遅い（全部待つ） | 速い（enqueue で返す） |
| 実装コスト | 低 | 高（retry/DLQ/監視） |
| 障害隔離 | 連鎖しやすい | 下流障害を吸収できる |
| 整合性 | 強い（トランザクション内） | 結果整合性 |
| デバッグ容易性 | スタックで追える | 分散トレーシング必須 |
| 向くワークロード | 残高照会、認証 | メール、画像処理、集計、Webhook |

## Cloud Run 固有の注意
- Push 型 Pub/Sub → Cloud Run は HTTP リクエストとして届く。**タイムアウト内に ack 返す**こと。重い処理は `min-instances` と `concurrency` を調整。
- CPU 常時割り当てを有効化しないと、リクエスト外で走る worker ロジックは止まる。
- Firebase Auth のトークン検証は Producer 側で終わらせ、イベントに `actor_id` を載せる（Consumer は再検証しない前提を明示）。

## チェックリスト
- [ ] Consumer は `message_id` または業務キーでべき等化されているか
- [ ] DB 書き込みと publish は Outbox で原子化されているか（デュアルライト禁止）
- [ ] 最大リトライ回数と DLQ・DLQ 監視アラートが設定されているか
- [ ] 「このキューが 1 時間止まってもユーザー体験は壊れないか」を説明できるか
- [ ] `processed_messages` テーブルに TTL or パーティション削除戦略があるか（無限増加防止）
