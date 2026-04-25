# 非同期処理・メッセージキューの設計パターン

## 概要

同期処理だけでシステムを構成すると、1箇所の遅延が全体に波及する。
非同期処理とメッセージキューは「処理の時間的結合」を切り離す設計手法。
重い処理（メール送信、画像処理、外部API連携）をリクエストから分離し、応答速度・耐障害性・スケーラビリティを同時に改善する。
AI時代では推論APIの呼び出し（数秒〜数十秒）を非同期化する場面が急増しており、必須知識。

## 仕組みの要点

### メッセージキューの基本構造

- **Producer** → **Queue/Topic** → **Consumer** の3要素
- Producerはメッセージを投げたら即座に返る（Fire & Forget）
- Consumerは自分のペースで処理（バックプレッシャー制御）
- キューがバッファとなり、Producer/Consumer間の速度差を吸収

### 主要パターン

| パターン | 特徴 | ユースケース |
|---|---|---|
| Point-to-Point | 1メッセージ→1Consumer | タスク処理、ジョブキュー |
| Pub/Sub | 1メッセージ→複数Subscriber | イベント通知、ログ集約 |
| Request-Reply | 非同期で応答を返す | 長時間処理の結果通知 |

### GCP構成での選択肢

- **Cloud Tasks**: HTTPターゲットへのタスク配信。リトライ・レート制御組み込み
- **Cloud Pub/Sub**: Pub/Subモデル。複数サービスへのイベント配信向き
- **Cloud Run Jobs**: バッチ的な重い処理の実行

## アンチパターン vs 正しい設計

| アンチパターン | 問題 | 正しい設計 |
|---|---|---|
| APIハンドラ内で重い処理を同期実行 | タイムアウト、UX劣化 | キューに投入し即レスポンス |
| べき等性を考慮しないConsumer | リトライで二重処理 | 処理IDで重複チェック |
| メッセージの順序に依存 | 順序保証はコストが高い | 各メッセージを独立・自己完結に |
| Dead Letter Queueを設定しない | 失敗メッセージが無限リトライ | DLQで隔離→後で調査・再処理 |
| キューの監視をしない | 滞留に気づかない | キュー深度・処理遅延のアラート設定 |

## コード例：FastAPI + Cloud Tasks

```python
# Producer側: APIハンドラからタスクを投入
from google.cloud import tasks_v2
from fastapi import FastAPI
import json

app = FastAPI()

@app.post("/orders")
async def create_order(order: OrderCreate):
    order_id = await save_order(order)  # DBに即保存
    # 重い処理はCloud Tasksに委譲
    client = tasks_v2.CloudTasksClient()
    task = {
        "http_request": {
            "http_method": "POST",
            "url": f"{WORKER_URL}/tasks/process-order",
            "headers": {"Content-Type": "application/json"},
            "body": json.dumps({"order_id": order_id}).encode(),
        }
    }
    client.create_task(parent=QUEUE_PATH, task=task)
    return {"order_id": order_id, "status": "accepted"}  # 即応答
```

```python
# Consumer側: べき等性を保証するワーカー
@app.post("/tasks/process-order")
async def process_order(payload: dict):
    order = await get_order(payload["order_id"])
    if order.status == "processed":  # べき等性チェック
        return {"status": "already_processed"}
    await execute_heavy_logic(order)
    await update_order_status(order.id, "processed")
    return {"status": "ok"}
```

## トレードオフ

| 観点 | 同期処理 | 非同期（キュー） |
|---|---|---|
| 実装の単純さ | シンプル | 複雑（Producer/Consumer分離） |
| 応答速度 | 処理完了まで待つ | 即応答（202 Accepted） |
| 障害の影響範囲 | 連鎖的 | キューで隔離される |
| デバッグ | スタックトレースで追える | 分散トレーシングが必要 |
| スケーリング | 全体を一律スケール | Consumer単独でスケール可能 |
| データ整合性 | トランザクションで保証 | 結果整合性（Eventual Consistency） |

## チェックリスト

- [ ] Consumer処理はべき等か（同じメッセージを2回受けても安全か）
- [ ] Dead Letter Queueを設定し、失敗メッセージの監視・再処理手順があるか
- [ ] キュー深度と処理遅延のモニタリング・アラートを設定したか
- [ ] メッセージサイズは制限内か（大きいデータはGCSに置きURLを渡す）
- [ ] Cloud Tasks/Pub SubへのIAM権限は最小限か（サービスアカウント分離）
