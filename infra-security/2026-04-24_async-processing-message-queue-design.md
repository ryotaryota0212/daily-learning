# 非同期処理・メッセージキューの設計パターン

## 概要

同期処理だけでシステムを構成すると、重い処理がレスポンスをブロックし、障害が連鎖する。
非同期処理とメッセージキューを適切に導入することで、**応答性・耐障害性・スケーラビリティ**を同時に改善できる。
AI時代では推論APIコールなど「遅くて不確実な外部呼び出し」が増えるため、非同期設計の重要性はさらに高まっている。

## 仕組みの要点

### 同期 vs 非同期の判断基準

| 同期が適切 | 非同期が適切 |
|---|---|
| 結果を即座に返す必要がある | 処理に数秒以上かかる |
| 処理が軽量（< 500ms） | 失敗時にリトライしたい |
| トランザクション整合性が必須 | 呼び出し元と処理を疎結合にしたい |
| ユーザーが結果を待っている | バックグラウンドで完了通知すればよい |

### 主要パターン

- **Task Queue（ワーカーモデル）**: APIがジョブを投入 → ワーカーが取り出して処理。Cloud Tasks向き
- **Pub/Sub（イベント駆動）**: 発行者はSubscriberを知らない。1対多の通知に最適
- **Event Sourcing**: 状態変更をイベントとして記録。監査ログや再構築が必要な場合に使う

### べき等性（Idempotency）が最重要

メッセージは**最低1回配信（at-least-once）** が基本。つまり重複実行される前提で設計する。

- リクエストに一意な `idempotency_key` を付与
- 処理前にDBで「処理済みか」を確認
- INSERT時は `ON CONFLICT DO NOTHING` を活用

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| API内で重い処理を同期実行 | ジョブをキューに投入し202を即返却 |
| リトライ時に重複処理が発生 | idempotency_keyで重複排除 |
| キュー消費者が1つだけ | ワーカー数を水平スケール可能に |
| 失敗メッセージを捨てる | Dead Letter Queue（DLQ）に退避 |
| キュー内のメッセージ順序に依存 | 順序非依存な設計、または明示的な順序制御 |

## コード例（FastAPI + Cloud Tasks）

```python
from fastapi import FastAPI, BackgroundTasks
from google.cloud import tasks_v2
import json, uuid

app = FastAPI()

@app.post("/orders", status_code=202)
async def create_order(order: OrderRequest):
    idempotency_key = str(uuid.uuid4())
    # DBにジョブ登録（状態: pending）
    await db.execute(
        "INSERT INTO jobs (key, status, payload) VALUES ($1, 'pending', $2)",
        idempotency_key, json.dumps(order.dict())
    )
    # Cloud Tasksにエンキュー
    client = tasks_v2.CloudTasksClient()
    task = {"http_request": {
        "url": f"{WORKER_URL}/process",
        "body": json.dumps({"key": idempotency_key}).encode(),
    }}
    client.create_task(parent=QUEUE_PATH, task=task)
    return {"job_id": idempotency_key, "status": "accepted"}
```

## トレードオフ

| 観点 | Cloud Tasks | Cloud Pub/Sub |
|---|---|---|
| ユースケース | 1対1のタスク実行 | 1対多のイベント配信 |
| 配信保証 | at-least-once | at-least-once |
| 遅延実行 | 対応（スケジュール可） | 非対応 |
| 順序保証 | なし | Ordering keyで可能 |
| 複雑さ | 低い | 中程度 |
| コスト | タスク数課金 | メッセージ量課金 |

## チェックリスト

- [ ] 重い処理（外部API・AI推論・メール送信）は非同期化されているか
- [ ] すべてのキュー消費処理にべき等性が担保されているか
- [ ] Dead Letter Queue（DLQ）を設定し、失敗メッセージを監視しているか
- [ ] ワーカーのタイムアウトとリトライ回数に上限を設けているか
- [ ] ジョブの状態（pending/processing/done/failed）をDBで追跡できるか
