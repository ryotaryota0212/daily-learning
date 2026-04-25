# 非同期処理・メッセージキューの設計パターン

## 概要

同期APIだけで構成されたシステムは、1箇所の遅延が全体に波及する。
非同期処理を導入することで、**応答速度・耐障害性・スケーラビリティ**を同時に改善できる。
AI時代では推論API呼び出し（数秒〜数十秒）が増え、非同期設計の重要性はさらに高まっている。
「どこを非同期にするか」「失敗したらどうするか」を設計できることが、システム設計力の核心。

## 仕組みの要点

### 同期 vs 非同期の判断基準

| 同期が適切 | 非同期が適切 |
|---|---|
| ユーザーが即座に結果を必要 | 処理に3秒以上かかる |
| 処理が軽量（< 500ms） | 失敗時にリトライしたい |
| トランザクション整合性が必須 | 複数サービスへの通知・連携 |
| 単純なCRUD | メール送信、PDF生成、AI推論 |

### 主要パターン

- **Point-to-Point（キュー）**: 1つのメッセージを1つのConsumerが処理。タスク実行向き
- **Pub/Sub（トピック）**: 1つのメッセージを複数のSubscriberが受信。イベント通知向き
- **イベント駆動**: 状態変化をイベントとして発行し、関心のあるサービスが反応

### べき等性（Idempotency）が最重要

- メッセージは**必ず重複配信される前提**で設計する（at-least-once delivery）
- 同じメッセージを2回処理しても結果が変わらないことを保証
- 実装方法: リクエストIDによる重複チェック、UPSERTの活用

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| 同期APIの中で重い処理を実行 | キューに投入し、非同期ワーカーで処理 |
| リトライ間隔が固定（1秒×10回） | Exponential backoff + jitter |
| 失敗メッセージを捨てる | Dead Letter Queue（DLQ）に退避 |
| メッセージ順序に依存した設計 | べき等性を担保し順序非依存にする |
| キューのConsumerが1つだけ | Consumer数をスケール可能に設計 |
| エラー時に無限リトライ | 最大リトライ回数 + DLQ + アラート |

## 設計例（FastAPI + Cloud Run + Cloud Tasks）

```python
# API側: 重い処理をCloud Tasksに委譲
@app.post("/api/reports/{report_id}/generate")
async def generate_report(report_id: str):
    task_client = tasks_v2.CloudTasksClient()
    task = {
        "http_request": {
            "http_method": "POST",
            "url": f"{WORKER_URL}/tasks/generate-report",
            "body": json.dumps({
                "report_id": report_id,
                "idempotency_key": str(uuid4()),
            }).encode(),
            "oidc_token": {"service_account_email": SA_EMAIL},
        }
    }
    task_client.create_task(parent=QUEUE_PATH, task=task)
    return {"status": "accepted", "report_id": report_id}  # 202
```

```python
# Worker側: べき等な処理
@app.post("/tasks/generate-report")
async def worker_generate(req: ReportTask):
    existing = await db.fetch_one(
        "SELECT status FROM reports WHERE id = :id", {"id": req.report_id}
    )
    if existing and existing["status"] == "completed":
        return {"status": "already_done"}  # べき等性の保証
    # 実際の生成処理...
    await db.execute(
        "UPDATE reports SET status = 'completed' WHERE id = :id",
        {"id": req.report_id},
    )
```

## GCP での選択肢

| サービス | 用途 | 特徴 |
|---|---|---|
| Cloud Tasks | タスクキュー | レート制御、スケジュール実行、リトライ設定 |
| Cloud Pub/Sub | イベント配信 | Pub/Sub型、複数Subscriber、高スループット |
| Cloud Scheduler | 定期実行 | cron的なジョブ。Cloud Runとの組合せ |
| Eventarc | イベントルーティング | GCPイベントをCloud Runに直接配信 |

**判断基準**: 1対1のタスク実行→Cloud Tasks、1対多のイベント通知→Pub/Sub

## トレードオフ

| 観点 | 同期処理 | 非同期処理 |
|---|---|---|
| 実装の単純さ | ◎ シンプル | △ 複雑になる |
| デバッグ容易性 | ◎ スタックトレースで追える | △ 分散トレーシング必要 |
| 応答速度 | △ 処理完了まで待つ | ◎ 即座に202返却 |
| 耐障害性 | △ 障害が連鎖 | ◎ キューがバッファになる |
| スケーラビリティ | △ 同時接続数に制約 | ◎ Consumer数で調整可能 |
| データ整合性 | ◎ トランザクションで保証 | △ 結果整合性を許容する設計 |

## チェックリスト

- [ ] 3秒以上かかる処理は非同期化を検討したか
- [ ] メッセージ処理はべき等に実装されているか
- [ ] Dead Letter Queue を設定し、アラートを繋いでいるか
- [ ] リトライはExponential backoff + 最大回数を設定しているか
- [ ] Worker（Consumer）を水平スケール可能な設計にしているか
