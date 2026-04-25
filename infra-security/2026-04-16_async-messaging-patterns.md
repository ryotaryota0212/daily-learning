# 非同期処理・メッセージキューの設計パターン

## 概要

同期的なリクエスト・レスポンスだけではシステムはスケールしない。
重い処理（メール送信、画像処理、外部API連携）を同期で行うとレイテンシが増大し、障害が連鎖する。
非同期処理はコンポーネント間を疎結合にし、負荷の平準化・障害の分離・スケーラビリティを実現する。
「どこを非同期にするか」の判断が、システム設計力の核心の一つ。

## 仕組みの要点

### メッセージングの基本モデル

- **Point-to-Point（キュー）**: 1つのメッセージを1つのコンシューマが処理。タスク分散に最適
- **Pub/Sub**: 1つのメッセージを複数のサブスクライバが受信。イベント通知に最適
- **Google Cloud Pub/Sub**: Cloud Run と相性が良く、Push型サブスクリプションでHTTPエンドポイントに配信可能

### 非同期化すべき処理の判断基準

- ユーザーが結果を即座に必要としない処理（メール送信、レポート生成）
- 外部サービス依存で失敗率が高い処理（決済、外部API呼び出し）
- 処理時間が長い（画像変換、データ集計、AI推論）
- 負荷が突発的に増える処理（イベント通知、バッチ処理）

### べき等性（Idempotency）が最重要

- メッセージは「少なくとも1回」配信される（At-Least-Once）ため、重複実行が前提
- **べき等キー**（リクエストIDなど）をDBに記録し、重複を検知する
- べき等でない処理を非同期にすると、二重課金・二重送信などの致命的バグになる

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| API内で重い処理を同期実行 | キューに投入し即座にレスポンス返却 |
| 失敗時にメッセージを捨てる | デッドレターキュー（DLQ）で失敗を保存 |
| リトライ間隔が固定 | 指数バックオフ（1s→2s→4s→8s） |
| べき等性を考慮しない | べき等キーで重複実行を防止 |
| 1つの巨大キューで全処理 | 処理種別ごとにトピック/キューを分離 |
| コンシューマの処理順序に依存 | 順序保証が必要なら同一Ordering Keyを使用 |

## コード/設計例

### FastAPI + Cloud Pub/Sub（タスク発行側）

```python
from google.cloud import pubsub_v1
import json, uuid

publisher = pubsub_v1.PublisherClient()
TOPIC = "projects/my-proj/topics/email-tasks"

@app.post("/orders/{order_id}/confirm")
async def confirm_order(order_id: str):
    idempotency_key = str(uuid.uuid4())
    publisher.publish(TOPIC, json.dumps({
        "order_id": order_id,
        "idempotency_key": idempotency_key,
        "action": "send_confirmation_email"
    }).encode())
    return {"status": "accepted", "idempotency_key": idempotency_key}
```

### Cloud Run Push サブスクリプション（処理側）

```python
@app.post("/workers/email")
async def process_email_task(request: Request):
    envelope = await request.json()
    data = json.loads(base64.b64decode(envelope["message"]["data"]))
    # べき等チェック
    if await db.fetchval("SELECT 1 FROM processed_tasks WHERE key=$1", data["idempotency_key"]):
        return {"status": "duplicate"}
    await send_email(data["order_id"])
    await db.execute("INSERT INTO processed_tasks(key) VALUES($1)", data["idempotency_key"])
    return {"status": "ok"}
```

## トレードオフ

| 観点 | 同期処理 | 非同期処理 |
|---|---|---|
| レイテンシ | 処理完了まで待つ | 即座にレスポンス |
| 複雑性 | 低い | 高い（リトライ、べき等性、DLQ） |
| デバッグ | 容易（スタックトレース） | 困難（分散トレーシング必要） |
| 障害分離 | 連鎖しやすい | 分離できる |
| スケール | ボトルネックになりやすい | コンシューマを独立スケール可能 |
| 整合性 | 強い一貫性 | 結果整合性（Eventually Consistent） |

## チェックリスト

- [ ] べき等性が保証されているか（重複メッセージで副作用が起きないか）
- [ ] デッドレターキュー（DLQ）を設定し、失敗メッセージを監視しているか
- [ ] リトライは指数バックオフ + 最大回数制限を設定しているか
- [ ] Cloud Run のタイムアウトとPub/Sub の Ack期限が整合しているか
- [ ] 非同期処理の結果をユーザーに伝える手段があるか（ポーリング/Webhook/通知）
