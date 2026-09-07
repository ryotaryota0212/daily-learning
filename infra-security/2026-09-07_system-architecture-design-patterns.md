# システム全体構成の設計パターン（API・DB・キャッシュ・キュー・CDN）

## 概要

「とりあえず動くコード」から「壊れにくいシステム」へ。AI時代においてコードを書く力より重要なのは、**構成要素をどう組み合わせるかを決める力**だ。API・DB・キャッシュ・キュー・CDNそれぞれの役割と配置を理解し、シンプルで拡張性ある全体像を設計できることが、エンジニアとしての差別化になる。

---

## 主要コンポーネントの役割と配置

### 基本レイヤー構造

```
[ユーザー]
    ↓
[CDN / Cloudflare]        ← 静的資産配信、DDoS防御、WAF
    ↓
[API Gateway / Cloud Run] ← 認証、レート制限、ルーティング
    ↓
[アプリサーバー]           ← ビジネスロジック
   ↙          ↘
[DB / Neon]   [Cache / Redis]  [Queue / Pub/Sub]
```

### 各コンポーネントの責務

| コンポーネント | 主な役割 | 使うべき場面 |
|---|---|---|
| CDN | 静的配信・エッジキャッシュ | 画像・JS・CSS |
| API Gateway | 認証・レート制限・ルーティング | 複数サービス統合時 |
| Cache (Redis) | 高速読み取り・セッション管理 | 読み込み頻度が高いデータ |
| Queue (Pub/Sub) | 非同期処理・負荷分散 | 時間のかかる処理・バースト対応 |
| DB (Neon/Postgres) | 永続化・整合性保証 | 状態を持つデータすべて |

---

## アンチパターン vs 正しい設計

### アンチパターン
- **DBに直接アクセスするフロント**: セキュリティホール・スケール困難
- **キャッシュなしのN+1クエリ**: DBがボトルネックになり全体が遅くなる
- **同期的に重い処理を実行**: レスポンスタイム増大・タイムアウトリスク
- **単一障害点（SPOF）**: DBやAPIが1台だと落ちたら全体停止
- **オーバーエンジニアリング**: 最初からマイクロサービス・Kafka導入

### 正しい設計の原則
- **シンプルに始める**: モノリス→必要になったら分割
- **読み書きを分離する**: 読み取りはCache・CDN、書き込みはDB
- **非同期化できる処理は非同期に**: Queue経由で切り離す
- **障害を局所化する**: コンポーネント間に疎結合・タイムアウトを設ける

---

## FastAPI + Neon + Cloud Run での実装例

```python
# キャッシュ戦略の実装例（Cache-Aside パターン）
import redis
from app.db import get_user_from_db

cache = redis.Redis(host="redis-host", port=6379, decode_responses=True)

async def get_user(user_id: str):
    # 1. キャッシュを確認
    cached = cache.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)
    
    # 2. DBから取得
    user = await get_user_from_db(user_id)
    
    # 3. キャッシュに書き込み（TTL: 300秒）
    cache.setex(f"user:{user_id}", 300, json.dumps(user))
    return user
```

```python
# 非同期処理のオフロード（Cloud Pub/Sub）
from google.cloud import pubsub_v1

publisher = pubsub_v1.PublisherClient()

async def send_notification(user_id: str, message: str):
    # キューに積んで即レスポンス（同期処理しない）
    topic_path = publisher.topic_path("project-id", "notifications")
    data = json.dumps({"user_id": user_id, "message": message}).encode()
    publisher.publish(topic_path, data)
    return {"status": "queued"}
```

---

## トレードオフ

| 選択肢 | メリット | デメリット |
|---|---|---|
| モノリス | 開発速度・シンプル | スケール限界・変更影響大 |
| マイクロサービス | 独立スケール・障害分離 | 複雑性・運用コスト大 |
| Redis追加 | 高速・DBへの負荷軽減 | 整合性管理・コスト増 |
| Pub/Sub追加 | 非同期・疎結合 | 遅延・デバッグ難しさ |
| CDNキャッシュ | 高速・DDoS耐性 | キャッシュ無効化の複雑さ |

**判断基準**: トラフィックが低いうちはシンプルに。ボトルネックが実際に現れてから追加する。

---

## チェックリスト

- [ ] 読み取り重視のデータにキャッシュを使っているか（DB直撃を避ける）
- [ ] 時間のかかる処理（メール送信・画像変換等）をキューで非同期化しているか
- [ ] 各コンポーネント間にタイムアウトとリトライを設定しているか
- [ ] SPOFを排除しているか（DBのフェイルオーバー・Cloud Runのオートスケール）
- [ ] CDNで静的資産をキャッシュし、オリジンへの負荷を下げているか
