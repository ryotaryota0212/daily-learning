# システム全体構成の設計パターン（API・DB・キャッシュ・キュー・CDN）

## 概要

「とりあえずDBに直接アクセスするAPIを作る」では壊れやすく、スケールしない。
システム設計の本質は「責務を分離し、各コンポーネントを適切に組み合わせること」。
AI時代には「コードを書く力」より「構成を決める力」が問われる。
FastAPI + Neon + Cloud Run のスタックでも同じ設計原則が適用できる。

---

## 仕組みの要点

### 主要コンポーネントの責務

| コンポーネント | 責務 | 例 |
|---|---|---|
| API層 | リクエスト受付・認証・バリデーション | Cloud Run + FastAPI |
| DB層 | 永続化・一貫性保証 | Neon (PostgreSQL) |
| キャッシュ層 | 読み取り高速化・DB負荷軽減 | Redis / Memorystore |
| キュー層 | 非同期処理・ピーク吸収 | Cloud Tasks / Pub/Sub |
| CDN層 | 静的アセット配信・エッジキャッシュ | Cloudflare |

### リクエストフロー（典型パターン）

```
Client
  → CDN（静的アセット・エッジキャッシュ）
  → API Gateway / Load Balancer（レート制限・TLS終端）
  → API Server (Cloud Run)
      → Cache (Redis): ヒットすればDBを叩かない
      → DB (Neon): キャッシュミス時 or 書き込み時
      → Queue: 時間のかかる処理を非同期化
```

---

## アンチパターン vs 正しい設計

### アンチパターン
- **全アクセスがDBに直行**：スケール不可・接続数上限に当たる
- **APIがビジネスロジックとDB操作を混在**：テスト不可・変更困難
- **同期処理で全て完結**：画像変換・メール送信をAPIレスポンスで待たせる
- **CDNなし**：全リクエストがオリジンに届く・レイテンシが高い

### 正しい設計
- **読み取りはキャッシュ → キャッシュミス時のみDB**（Cache-Aside）
- **重い処理はキューに積んで202 Accepted を即返す**
- **静的コンテンツはCDNで配信・APIはオリジンのみ**
- **APIはステートレス → 水平スケールを容易にする**

---

## コード/設計例（最小限）

### FastAPI でのキャッシュ + 非同期キューパターン

```python
# Cache-Aside + 非同期処理の組み合わせ
@router.get("/users/{user_id}")
async def get_user(user_id: str, cache: Redis = Depends(get_cache), db = Depends(get_db)):
    cached = await cache.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)
    user = await db.fetchrow("SELECT * FROM users WHERE id=$1", user_id)
    await cache.setex(f"user:{user_id}", 300, json.dumps(dict(user)))
    return user

@router.post("/reports")
async def generate_report(task_queue = Depends(get_queue)):
    task_id = str(uuid4())
    await task_queue.enqueue("generate_report", task_id=task_id)
    return {"task_id": task_id, "status": "accepted"}  # 202 Accepted
```

---

## トレードオフ

| 判断軸 | シンプル構成 | フル構成 |
|---|---|---|
| 開発速度 | 速い | 遅い（コンポーネント増加） |
| 運用コスト | 低い | 高い（Redis・Queue管理） |
| スケーラビリティ | 低い | 高い |
| 障害点 | 少ない | 多い（各コンポーネントが障害源になりうる） |

**原則：スケールが必要になってから追加する。最初から全部入りにしない。**
- 月間10万リクエスト以下 → キャッシュ不要の場合が多い
- DB接続数が上限に近づいたら → キャッシュ導入を検討
- APIレスポンスタイムが1秒を超える処理が出たら → キュー導入を検討

---

## チェックリスト

- [ ] 各コンポーネントの責務が明確か（APIがDBロジックを持っていないか）
- [ ] 重い処理（メール・画像・レポート生成）は非同期化されているか
- [ ] 読み取り頻度が高いデータにキャッシュ層があるか
- [ ] APIがステートレスで水平スケール可能か（セッションをDBやRedisで管理）
- [ ] 各コンポーネントが落ちたとき何が起きるか説明できるか
