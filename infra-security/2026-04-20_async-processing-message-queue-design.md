# 非同期処理・メッセージキューの設計パターン

## 概要

同期処理だけのシステムは「一箇所が遅いと全体が遅い」という脆さを持つ。
非同期処理を導入すると、重い処理を切り離してレスポンスを高速化し、障害の連鎖を防げる。
AI時代では推論APIの呼び出し等、秒単位の外部処理が増えるため、非同期設計の重要性はさらに高い。
「どこを非同期にするか」「失敗したらどうするか」を設計できることが、システムの信頼性を決める。

## 仕組みの要点

### 同期 vs 非同期の使い分け

| 条件 | 同期 | 非同期 |
|------|------|--------|
| ユーザーが即座に結果を必要とする | o | - |
| 処理に3秒以上かかる | - | o |
| 失敗してもリトライで回復可能 | - | o |
| 外部API呼び出しが不安定 | - | o |
| 処理順序の厳密な保証が必要 | o | - |

### 主要パターン

- **Fire-and-Forget**: タスクを投げて結果を待たない（ログ送信、メトリクス記録）
- **Request-Reply（非同期）**: ジョブIDを返し、クライアントがポーリングまたはWebhookで結果取得
- **Pub/Sub**: 1つのイベントを複数のサブスクライバーが独立に処理（注文→在庫更新、メール送信、ログ）
- **イベント駆動**: 状態変化をイベントとして発行し、後続処理をトリガー

### べき等性（最重要概念）

メッセージは「少なくとも1回」配信される（at-least-once）。つまり**重複実行が前提**。

- リクエストにべき等性キー（idempotency key）を含める
- DB操作は `INSERT ... ON CONFLICT DO NOTHING` または `UPDATE WHERE` で重複を吸収
- 処理済みイベントIDを記録して重複チェック

## アンチパターン vs 正しい設計

| アンチパターン | 問題 | 正しい設計 |
|---------------|------|-----------|
| HTTP内で重い処理を同期実行 | タイムアウト、リソース枯渇 | Cloud Tasksにオフロード |
| リトライ時に同じ処理が二重実行 | データ不整合 | べき等性キーで重複排除 |
| 失敗メッセージを無限リトライ | キュー詰まり（poison pill） | Dead Letter Queue（DLQ）に退避 |
| キュー障害で全機能停止 | 単一障害点 | グレースフルデグラデーション設計 |
| メッセージ順序に依存したロジック | 順序保証なしで破綻 | 各メッセージを自己完結型にする |

## コード例：Cloud Tasks による非同期オフロード（FastAPI + Cloud Run）

```python
# エンドポイント: ジョブ受付（即座にレスポンス）
@app.post("/reports")
async def create_report(req: ReportRequest, user=Depends(get_current_user)):
    job_id = str(uuid4())
    await db.execute(
        "INSERT INTO jobs (id, user_id, status) VALUES ($1, $2, 'pending')",
        job_id, user.uid
    )
    # Cloud Tasksにエンキュー（別のCloud Runエンドポイントを呼ぶ）
    enqueue_task("/workers/generate-report", {"job_id": job_id})
    return {"job_id": job_id, "status": "pending"}

# ワーカー: 実際の重い処理（べき等性を担保）
@app.post("/workers/generate-report")
async def worker_generate_report(payload: dict):
    job = await db.fetchrow("SELECT * FROM jobs WHERE id = $1", payload["job_id"])
    if job["status"] == "completed":  # べき等性: 処理済みならスキップ
        return {"ok": True}
    result = await heavy_report_generation(job)
    await db.execute(
        "UPDATE jobs SET status='completed', result=$1 WHERE id=$2 AND status='pending'",
        result, job["id"]  # WHERE status='pending' で二重更新を防止
    )
```

## トレードオフ

| 選択肢 | メリット | デメリット | 適用場面 |
|--------|---------|-----------|---------|
| Cloud Tasks | GCP統合、リトライ自動、簡単 | 順序保証なし、ペイロード1MB制限 | Cloud Run間の非同期呼び出し |
| Cloud Pub/Sub | 高スループット、Fan-out可能 | 順序保証は追加設定要 | イベント駆動、複数サブスクライバー |
| Cloud Workflows | 複雑なフロー制御、可視化 | 学習コスト、レイテンシ | マルチステップの長時間処理 |
| DB polling | インフラ追加不要、シンプル | スケールしにくい、遅延あり | 小規模、キュー導入前の初期段階 |

## チェックリスト

- [ ] 非同期化した処理は**べき等**に設計されているか
- [ ] Dead Letter Queue（DLQ）を設定し、失敗メッセージの監視・アラートがあるか
- [ ] ジョブのステータスをユーザーが確認できるAPI（ポーリング or Webhook）があるか
- [ ] Cloud Runワーカーのタイムアウト設定がジョブの最大実行時間を超えているか
- [ ] キューが詰まった場合のグレースフルデグラデーション（同期フォールバック等）を検討したか
