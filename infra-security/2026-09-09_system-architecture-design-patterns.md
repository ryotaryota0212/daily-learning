# システム全体構成の設計パターン（API・DB・キャッシュ・キュー・CDN）

## 概要

「とりあえず動く」段階を超えて、スケールと障害に耐えるシステムを設計するには、各レイヤーの役割と連携を俸じることが必要だ。AI時代においてコードを書く力より「どのコンポーネントをどう組み合わせるか」の判断力が価値を持つ。本記事では FastAPI + Neon + Cloud Run + Firebase Auth スタックを軸に、典型的な設計パターンとトレードオフを整理する。

---

## 仕組みの要点

### レイヤー構成（外側から内側へ）

```
Client
  └── CDN / Edge（Cloudflare）          ← 静的配信・WAF・DDoS防御
        └── API Gateway / Load Balancer  ← ルーティング・認証・レート制限
              └── App Server（Cloud Run） ← ビジネスロジック
                    ├── Cache（Redis）    ← 頻繁な読み取りを吸収
                    ├── Queue（Pub/Sub）  ← 非同期・重い処理を分離
                    └── DB（Neon/Postgres）← 真実の源泉
```

### 各コンポーネントの責務

- **CDN**: 静的ファイル・APIレスポンスのエッジキャッシュ。RTT削減とオリジン負荷軽減
- **API Gateway**: 認証（Firebase Auth トークン検証）、レート制限、ルーティング
- **App Server (Cloud Run)**: ステートレスに保つ。スケールアウト前提
- **Cache (Redis)**: 計算コストが高い・変化が少ないデータを一時保存。TTL必須
- **Queue (Pub/Sub)**: メール送信・画像変換など即時応答不要な処理を分離
- **DB (Neon)**: 永続化の唯一の真実。RLS で行レベルセキュリティ

---

## アンチパターン vs 正しい設計

| アンチパターン | 問題 | 正しい設計 |
|---|---|---|
| App Server が DB に直接大量クエリ | N+1 問題、DB過負荷 | キャッシュ層を挟む / バッチ取得 |
| 同期で重い処理（メール送信など） | レスポンス遅延、タイムアウト | Queue に流して非同期化 |
| App Server にセッション状態を持つ | スケールアウト不可 | ステートレス化・セッションはRedis |
| DB を唯一の通信路にする | DB がボトルネック | 読み取りはキャッシュ、書き込みのみDB |
| CDN なしで全リクエストをオリジンへ | 帯域・コスト・速度の問題 | 静的資産・APIは積極的にCDNキャッシュ |

---

## 設計例：FastAPI + Cloud Run の典型構成

```python
# FastAPI: ステートレス設計の基本
from fastapi import FastAPI, Depends
from redis.asyncio import Redis

app = FastAPI()
redis: Redis = None  # 起動時に接続、インスタンス間で共有しない

async def get_user_profile(user_id: str, redis: Redis) -> dict:
    cache_key = f"user:{user_id}:profile"
    cached = await redis.get(cache_key)
    if cached:
        return json.loads(cached)
    
    profile = await db.fetch_one("SELECT * FROM users WHERE id = $1", user_id)
    await redis.setex(cache_key, 300, json.dumps(profile))  # TTL 5分
    return profile

# 重い処理はQueueへ（Cloud Pub/Sub）
async def trigger_report_generation(user_id: str):
    await pubsub_client.publish(topic="reports", data={"user_id": user_id})
    return {"status": "queued"}  # 即座に返す
```

---

## トレードオフ

| 選択 | メリット | デメリット |
|---|---|---|
| キャッシュを厚くする | 高速・DB負荷軽減 | データ陳腐化リスク・無効化の複雑さ |
| キューを使う | 非同期でスループット向上 | 結果整合性・デバッグ難度が上がる |
| CDNキャッシュを積極活用 | 低レイテンシ・コスト削減 | キャッシュパージの設計が必要 |
| Cloud Run（サーバーレス） | スケールゼロ・運用コスト低 | コールドスタート・最大並行数制限 |
| Neon（サーバーレス Postgres） | ブランチ機能・スケール | 接続数制限・コネクションプールが必須 |

---

## チェックリスト

- [ ] App Server はステートレスか（セッション状態を持っていないか）
- [ ] 読み取り頻度が高いデータにキャッシュ（+ TTL）を設定しているか
- [ ] 即時応答不要な処理（メール・バッチ）をキューで非同期化しているか
- [ ] DB への接続はコネクションプール経由か（Neon は PgBouncer 推奨）
- [ ] 各コンポーネントが落ちたとき何が起きるか想定できているか
