# 非同期処理・メッセージキューの設計パターン

## 概要

同期処理だけでシステムを組むと、1箇所の遅延が全体に波及する。
非同期処理は「今すぐ結果がいらない処理」を切り離し、システムの応答性・耐障害性・スケーラビリティを同時に改善する手段。
AI時代では推論API呼び出し（数秒〜数十秒）が増え、非同期設計の重要度はさらに上がっている。
「どの処理を非同期にするか」「失敗したらどうするか」を設計できることが、システムを成立させる力の核心。

## 仕組みの要点

### 同期 vs 非同期の判断基準

| 同期が適切 | 非同期が適切 |
|---|---|
| ユーザーが即座に結果を必要とする | 結果を後で通知すれば十分 |
| 処理が100ms以内に完了 | 処理が数秒以上かかる |
| 失敗時に即座にリトライ可能 | 失敗時に再試行キューで対応 |
| 例: ログイン認証 | 例: メール送信、PDF生成、AI推論 |

### 主要パターン

- **Fire-and-Forget**: タスクを投げて結果を待たない（ログ送信、Analytics）
- **Request-Reply（非同期）**: タスクIDを返し、ポーリングまたはWebhookで結果通知
- **Pub/Sub**: 1つのイベントを複数のサブスクライバーが独立に処理（注文確定→在庫更新、メール送信、ログ記録）
- **Work Queue**: 複数ワーカーがキューからタスクを取得し分散処理

### べき等性（Idempotency）が最重要

メッセージは「最低1回配信（at-least-once）」が基本。つまり**同じ処理が2回実行されても結果が変わらない**設計が必須。

- DBのUPSERT（INSERT ON CONFLICT）を使う
- 処理済みIDをテーブルに記録し、重複チェック
- 外部API呼び出しにはidempotency keyを付与

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| API内で重い処理を同期実行し、タイムアウト | Cloud Tasksに投げて即202レスポンス |
| 失敗時にリトライ無制限 | 指数バックオフ + 最大リトライ回数 + Dead Letter Queue |
| メッセージ順序に依存したロジック | 各メッセージをべき等に設計し順序非依存に |
| キュー消費者が1つだけ（SPOF） | 複数ワーカー + 自動スケール |
| 失敗メッセージを捨てる | DLQに退避し、調査・再処理可能に |

## コード/設計例（FastAPI + Cloud Tasks）

```python
# API側: 重い処理をCloud Tasksに委譲
@app.post("/api/reports", status_code=202)
async def create_report(req: ReportRequest, user=Depends(get_current_user)):
    task_id = str(uuid4())
    await db.execute(
        "INSERT INTO tasks (id, user_id, status) VALUES ($1, $2, 'pending')",
        task_id, user.uid
    )
    # Cloud Tasksにエンキュー（リトライは自動）
    enqueue_cloud_task("/worker/generate-report", {"task_id": task_id})
    return {"task_id": task_id, "status": "pending"}

# Worker側: べき等な処理
@app.post("/worker/generate-report")
async def worker_generate_report(payload: dict):
    task = await db.fetchrow("SELECT * FROM tasks WHERE id = $1", payload["task_id"])
    if task["status"] == "completed":
        return {"ok": True}  # べき等: 処理済みならスキップ
    result = await heavy_report_generation(task)
    await db.execute(
        "UPDATE tasks SET status='completed', result=$1 WHERE id=$2",
        result, task["id"]
    )
```

## GCP スタックでの選択指針

| サービス | 用途 | 特徴 |
|---|---|---|
| Cloud Tasks | 1対1のタスク実行 | リトライ制御、レート制限、スケジュール |
| Cloud Pub/Sub | 1対多のイベント配信 | Fan-out、フィルタリング、DLQ |
| Cloud Scheduler | 定期実行 | cron的ジョブ、Cloud Runと連携 |
| Workflows | 複数ステップの処理連携 | オーケストレーション、条件分岐 |

## トレードオフ

| 観点 | 同期処理 | 非同期処理 |
|---|---|---|
| 実装の単純さ | シンプル | 状態管理が必要で複雑 |
| ユーザー体験 | 即座にレスポンス | ポーリング/通知の実装が必要 |
| 耐障害性 | 1箇所の障害が全体に波及 | キューがバッファとなり障害を吸収 |
| スケーラビリティ | ボトルネックで全体が詰まる | ワーカーを独立にスケール可能 |
| デバッグ | スタックトレースで追える | 分散トレーシングが必要 |
| データ整合性 | トランザクションで保証 | 結果整合性、べき等性の設計が必要 |

## チェックリスト

- [ ] この処理は本当に非同期にすべきか？（100ms以内なら同期で十分）
- [ ] ワーカーの処理はべき等か？（同じメッセージが2回来ても安全か）
- [ ] 失敗時のリトライ戦略とDead Letter Queueは設定したか？
- [ ] タスクの状態を追跡でき、ユーザーに進捗を伝えられるか？
- [ ] ワーカーが停止・遅延した場合のアラートは設定したか？
