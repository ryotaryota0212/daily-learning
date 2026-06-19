# システム全体構成の設計パターン（API・DB・キャッシュ・キュー・CDN）

## 概要

「とりあえず動く」から「スケールして壊れにくい」へ移行するには、コンポーネントの責務分担を最初に正しく決める必要がある。  
AI時代において、このアーキテクチャ判断こそが差別化になる。  
「どこにキャッシュを置くか」「どこを非同期にするか」「どこがSPOFになるか」を答えられることがゴール。

---

## 基本レイヤー構成

```
ユーザー
  └─ CDN（Cloudflare） ← 静的ファイル・DDoS防御
       └─ API Gateway / Load Balancer
            └─ Cloud Run（FastAPI）← ステートレス
                 ├─ Neon PostgreSQL ← 永続データ
                 ├─ Redis（Upstash）← キャッシュ・セッション・レート制限
                 └─ Cloud Pub/Sub  ← 非同期ジョブ
```

### 各コンポーネントの責務

| コンポーネント | 責務 | 置かない方がよいもの |
|---|---|---|
| CDN | 静的配信・WAF・TLS終端 | ビジネスロジック |
| API（Cloud Run） | リクエスト処理・認可 | 状態・長時間処理 |
| DB（Neon） | 永続化・トランザクション | キャッシュ・集計専用 |
| Cache（Redis） | 高速読み取り・TTL付き一時データ | 消えてはいけないデータ |
| Queue（Pub/Sub） | 非同期・疎結合・リトライ | 即時応答が必要な処理 |

---

## アンチパターン vs 正しい設計

### アンチパターン
- DBに直接アクセスする処理を複数サービスが持つ → 結合度が上がり障害が伝播
- キャッシュなしでDBをフルスキャンするAPIを高頻度で呼ぶ
- 重い処理（メール送信・AI推論）を同期で実行しタイムアウト多発
- CDNを挟まず全トラフィックがCloud Runに到達する
- セッション状態をCloud Runメモリに持つ（スケールアウトで壊れる）

### 正しい設計
- 読み取り多 → Cache-Aside でRedisを前段に置く
- 重い処理 → Pub/Sub でキューに積み、Workerが非同期処理
- スパイク対策 → CDNキャッシュ + Cloud Run auto-scaling
- ステートレス維持 → セッションはRedis、ファイルはGCS

---

## 設計例：FastAPI + Neon + Redis + Pub/Sub

```python
# キャッシュ付き読み取り（Cache-Aside）
async def get_user(user_id: str) -> User:
    cached = await redis.get(f"user:{user_id}")
    if cached:
        return User.parse_raw(cached)
    user = await db.fetch_one("SELECT * FROM users WHERE id=$1", user_id)
    await redis.setex(f"user:{user_id}", 300, user.json())  # TTL 5分
    return user

# 重い処理は非同期キューへ
async def request_report(user_id: str, params: dict):
    await pubsub_client.publish("report-jobs", {
        "user_id": user_id, "params": params, "idempotency_key": str(uuid4())
    })
    return {"status": "queued"}  # 即時レスポンス
```

---

## トレードオフ

| 判断軸 | 選択肢A | 選択肢B | 基準 |
|---|---|---|---|
| DB vs Cache | DBに毎回問い合わせ | Redis前段 | QPS > 100/s ならキャッシュ |
| 同期 vs 非同期 | APIで直接実行 | Queue + Worker | 処理が200ms超えるなら非同期 |
| モノリス vs 分割 | 単一Cloud Runサービス | 複数サービス | 最初はモノリス、負荷点が明確になってから分割 |
| CDNキャッシュ | エッジキャッシュする | オリジンに都度到達 | 認証不要な静的・準静的データはCDN |
| スケール方向 | 垂直（インスタンス強化） | 水平（台数増加） | Cloud Runは水平が基本 |

---

## チェックリスト

- [ ] 各コンポーネントがSPOF（単一障害点）になっていないか確認した
- [ ] 重い処理（AI推論・メール・帳票）はキューに逃がしている
- [ ] Cloud Runインスタンスはステートレス（セッションをRedisに外出し）
- [ ] DBへの高頻度クエリの前にキャッシュ層を挟んでいる
- [ ] CDNでTLS終端・静的ファイルキャッシュを設定している
