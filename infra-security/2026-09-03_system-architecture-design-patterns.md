# システム全体構成の設計パターン（API・DB・キャッシュ・キュー・CDN）

## 概要

「動くコードを書く」より「システム全体をどう組むか」を決める力がAI時代に最も重要。
APIサーバー・DB・キャッシュ・メッセージキュー・CDNのそれぞれの役割と連携パターンを理解していないと、
スケールしない・壊れやすい・コスト爆発する設計になる。
正しい構成の選択は、機能開発よりも先に考えるべき基盤の意思決定である。

---

## 仕組みの要点

### 主要コンポーネントの役割

| コンポーネント | 役割 | 使用例 |
|---|---|---|
| **API層** (Cloud Run) | リクエスト処理・ビジネスロジック | FastAPI エンドポイント |
| **DB** (Neon/PostgreSQL) | 永続データの読み書き | ユーザー・注文データ |
| **キャッシュ** (Redis/Memcached) | 高速読み取り・DB負荷軽減 | セッション・集計結果 |
| **メッセージキュー** (Pub/Sub) | 非同期処理・疎結合 | メール送信・集計バッチ |
| **CDN** (Cloudflare) | 静的コンテンツ配信・DDoS防御 | 画像・JS・キャッシュHTTP |

### 典型的なリクエストフロー

```
Client → CDN → Load Balancer → API (Cloud Run)
                                  ↓        ↓
                               Cache     DB (Neon)
                                  ↓
                             Message Queue → Worker
```

### 設計の大原則

- **読み取り多 → キャッシュ優先**: 書き込みより読み取りが多い場合は積極的にキャッシュ
- **重い処理 → 非同期化**: レスポンスに不要な処理はキューに投げる
- **静的コンテンツ → CDNオフロード**: APIサーバーに静的ファイルを返させない
- **DB接続数 → コネクションプール必須**: Cloud RunはサーバーレスなのでPgBouncer等が必要

---

## アンチパターン vs 正しい設計

### アンチパターン

- **全部DBから取る**: セッションや集計もDBに問い合わせ → 遅延増大・DB過負荷
- **同期で重い処理**: メール送信・PDF生成をリクエスト内で実行 → タイムアウト
- **APIが静的ファイルを返す**: Cloud Runで画像を配信 → コスト高・遅い
- **接続プールなし**: Cloud Runの各インスタンスがDB直接接続 → 接続上限超過

### 正しい設計

- セッション・一時データ: Redis/Memcached
- 集計・マスタデータ: Cache-Aside パターン（DBからロード→キャッシュ保存）
- 重い処理: Pub/Sub でワーカーに委譲
- 静的リソース: Cloud Storage + Cloudflare CDN
- DB接続: PgBouncer or Neon接続プール設定

---

## コード/設計例（FastAPI + Cloud Run構成）

```python
# app/dependencies.py - キャッシュ優先のデータ取得パターン
import redis.asyncio as redis
from app.db import get_db

cache = redis.Redis(host="redis-host", decode_responses=True)

async def get_user_profile(user_id: str):
    cache_key = f"user:{user_id}"
    cached = await cache.get(cache_key)
    if cached:
        return json.loads(cached)

    async with get_db() as db:
        user = await db.fetchrow("SELECT * FROM users WHERE id=$1", user_id)

    await cache.setex(cache_key, 300, json.dumps(dict(user)))  # TTL 5分
    return dict(user)

# 重い処理を非同期キューへ
from google.cloud import pubsub_v1

publisher = pubsub_v1.PublisherClient()

async def send_welcome_email(user_id: str, email: str):
    topic = "projects/my-project/topics/email-queue"
    data = json.dumps({"user_id": user_id, "email": email}).encode()
    publisher.publish(topic, data)  # 非同期で返す。ワーカーが処理
```

---

## トレードオフ

| 構成 | メリット | デメリット |
|---|---|---|
| キャッシュ追加 | 高速・DB負荷減 | データ不整合リスク・管理コスト |
| メッセージキュー | 疎結合・耐障害性 | 複雑性増加・遅延 |
| CDN | 高速・コスト減 | キャッシュ制御が難しい |
| マイクロサービス | 独立スケール | 運用コスト・ネットワーク遅延 |

**スタートアップ期の推奨**: モノリス API + DB + CDN + 必要な部分だけキュー。
早期のマイクロサービス化は複雑性を増やすだけで逆効果。

---

## チェックリスト

- [ ] DBへの接続はコネクションプールを経由しているか（Cloud Run ↔ Neon）
- [ ] 繰り返し読む重いクエリはキャッシュしているか
- [ ] タイムアウトしうる処理（メール・PDF・外部API）は非同期化したか
- [ ] 静的ファイルはCDN配信し、APIサーバーに返させていないか
- [ ] キャッシュのTTLとキャッシュ無効化の戦略を明示的に決めているか
