# 非同期処理・メッセージキューの設計パターン

## 概要

同期処理だけでシステムを組むと、1箇所の遅延が全体を止める。
非同期処理とメッセージキューは「処理の時間的分離」を実現し、スケーラビリティと耐障害性を高める。
AI時代では推論APIの呼び出し（数秒〜数十秒）が典型的なユースケースであり、非同期設計の重要性は増している。
「どこを非同期にするか」「失敗したらどうなるか」を設計できることが、システム設計力の核心。

## 仕組みの要点

### 同期 vs 非同期の判断基準

| 同期が適切 | 非同期が適切 |
|---|---|
| ユーザーが即座に結果を必要とする | 処理に3秒以上かかる |
| 処理が軽量（< 500ms） | 失敗時にリトライしたい |
| トランザクション整合性が必須 | 複数サービスに通知が必要 |
| 依存関係が直線的 | スパイク負荷を平準化したい |

### 主要パターン

- **Fire-and-Forget**: タスクを投げて完了を待たない（ログ送信、メトリクス記録）
- **Request-Reply（非同期）**: ジョブIDを返し、クライアントがポーリングまたはWebhookで結果取得
- **Pub/Sub**: 1つのイベントを複数のサブスクライバーが独立に処理（注文確定→在庫更新、メール送信、分析）
- **イベント駆動**: 状態変化をイベントとして発行し、下流が反応する

### べき等性（Idempotency）が最重要

メッセージは**最低1回（at-least-once）** 配信が基本。つまり重複実行される前提で設計する。

- リクエストに一意な `idempotency_key` を付与
- 処理前にキーの存在チェック → 存在すれば既存結果を返す
- DB操作は `INSERT ... ON CONFLICT DO NOTHING` パターン

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| API内で重い処理を同期実行し、タイムアウト | ジョブキューに投入し、ジョブIDを即座に返す |
| メッセージ消費失敗時に即座に破棄 | Dead Letter Queue（DLQ）に退避し、後で調査・再処理 |
| べき等性を考慮せず、重複実行で二重課金 | idempotency_keyで重複チェック |
| キューの消費者が1つだけでボトルネック | コンシューマーの水平スケール＋パーティショニング |
| 非同期にした結果、エラーを誰も見ていない | DLQ監視＋アラート設定 |
| すべてを非同期にして複雑化 | 同期で十分な処理は同期のまま |

## 設計例（FastAPI + Cloud Tasks）

```python
# エンドポイント: ジョブ投入（即座にジョブIDを返す）
@app.post("/reports", status_code=202)
async def create_report(req: ReportRequest):
    job_id = str(uuid4())
    await db.execute(
        "INSERT INTO jobs (id, status) VALUES ($1, 'pending')", job_id
    )
    await enqueue_task("/worker/generate-report", {"job_id": job_id, **req.dict()})
    return {"job_id": job_id, "status": "pending"}

# ワーカー: べき等な処理
@app.post("/worker/generate-report")
async def worker(payload: dict):
    job = await db.fetchrow("SELECT * FROM jobs WHERE id=$1", payload["job_id"])
    if job["status"] == "completed":
        return  # べき等: 処理済みならスキップ
    result = await generate_report(payload)
    await db.execute(
        "UPDATE jobs SET status='completed', result=$1 WHERE id=$2",
        result, payload["job_id"]
    )
```

## GCP での実装選択肢

| サービス | ユースケース | 特徴 |
|---|---|---|
| **Cloud Tasks** | 1対1の非同期タスク実行 | リトライ制御、レート制限、Cloud Runと統合容易 |
| **Cloud Pub/Sub** | 1対多のイベント配信 | Fan-out、順序保証オプション、DLQサポート |
| **Cloud Scheduler** | 定期実行ジョブ | cron式、Cloud Run/Pub/Subへ発火 |
| **Eventarc** | GCPイベント駆動 | GCSアップロード→Cloud Run起動等 |

**FastAPI + Cloud Run 構成での推奨**: Cloud Tasks（タスクキュー）+ Pub/Sub（イベント配信）の併用

## トレードオフ

| 観点 | 同期処理 | 非同期処理 |
|---|---|---|
| 実装の単純さ | ◎ シンプル | △ 状態管理が必要 |
| レイテンシ | ◎ 即座に結果 | △ 結果取得に別リクエスト |
| スケーラビリティ | △ 直列ボトルネック | ◎ キューで平準化 |
| 障害耐性 | △ 下流障害が即伝播 | ◎ キューがバッファ |
| デバッグ容易性 | ◎ スタックトレース1本 | △ 分散トレーシング必要 |
| データ整合性 | ◎ トランザクション容易 | △ 結果整合性が前提 |

## チェックリスト

- [ ] 3秒以上かかる処理を同期APIで実行していないか
- [ ] メッセージ処理はべき等に実装されているか（重複実行しても安全か）
- [ ] Dead Letter Queue を設定し、失敗メッセージの監視・アラートがあるか
- [ ] コンシューマーの水平スケールが可能な設計か
- [ ] 非同期ジョブの状態をクライアントが確認できる手段（ポーリング/Webhook）があるか
