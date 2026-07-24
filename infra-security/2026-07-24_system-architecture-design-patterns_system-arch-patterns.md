# システム全体構成の設計パターン（API・DB・キャッシュ・キュー・CDN）

## 概要

「動くコードを書く力」より「システムを成立させる力」が問われる時代に、最も基本的なスキルがシステム全体構成の設計だ。
APIサーバー・DB・キャッシュ・メッセージキュー・CDNという5つのレイヤーの役割と接続関係を正しく理解すれば、スケーラビリティ・可用性・コストのすべてが変わる。
なぜどのコンポーネントを使うかを説明できることが、AI時代のエンジニアに求められるコアスキルである。

---

## 仕組みの要点

### 典型的な構成レイヤー

```
ユーザー
  ↓ HTTPS
CDN（Cloudflare）        ← 静的ファイル・エッジキャッシュ
  ↓
API Gateway / Load Balancer
  ↓
API Server（Cloud Run）  ← ビジネスロジック・認証
  ↓            ↓
Cache         Message Queue
(Redis)       (Pub/Sub)
  ↓            ↓
Database      Worker（Cloud Run）
(Neon/PG)
```

### 各コンポーネントの責務

| コンポーネント | 役割 | 何を置かないか |
|---|---|---|
| CDN | 静的資産配信・DDoS吸収 | 個人データ・認証必須コンテンツ |
| API Server | ビジネスロジック・認証検証 | 重い計算・長時間処理 |
| Cache (Redis) | 高速な読み取り・セッション管理 | 唯一の真実（Source of Truth） |
| Message Queue | 非同期処理・負荷平準化 | 即時応答が必要な処理 |
| Database | 永続化・トランザクション | 全文検索・大量集計 |

---

## アンチパターン vs 正しい設計

### アンチパターン

- **APIサーバーが全てを持つ**: DB直叩き・キャッシュなし・同期処理の三重苦
- **DBをキャッシュ代わりに使う**: `SELECT * WHERE id = ?` を毎リクエスト叩く
- **同期で全部やろうとする**: メール送信・画像変換をAPIレスポンス内で処理
- **CDNに動的レスポンスをキャッシュ**: 認証済みデータが別ユーザーに見える

### 正しい設計の原則

- **読み取りはキャッシュファースト**: DB負荷を1/10以下にする
- **重い処理はキューに流す**: APIは「受け付けた」だけ返してWorkerに委譲
- **CDNは公開・静的のみ**: `Cache-Control: private` で個人データを保護
- **DBは書き込み・複雑クエリ・トランザクションだけ**: 単純な読み取りはRedisで完結

---

## コード/設計例（FastAPI + Redis + Pub/Sub）

```python
# API: キャッシュファースト + 非同期キューへの委譲
@app.get("/users/{user_id}/report")
async def get_report(user_id: str, redis: Redis, pubsub: PubSubClient):
    # 1. キャッシュヒットなら即返す
    cached = await redis.get(f"report:{user_id}")
    if cached:
        return {"status": "ready", "data": json.loads(cached)}

    # 2. ミスならWorkerに投げてAccepted返す
    await pubsub.publish("generate-report", {"user_id": user_id})
    return {"status": "processing"}, 202

# Worker: 重い処理はここで
async def handle_generate_report(event):
    report = await generate_heavy_report(event["user_id"])
    await redis.setex(f"report:{event['user_id']}", 3600, json.dumps(report))
```

---

## トレードオフ

| 選択 | メリット | デメリット |
|---|---|---|
| Redisキャッシュ追加 | レイテンシ大幅改善・DB負荷減 | データ整合性管理が必要・追加コスト |
| メッセージキュー追加 | APIレスポンス高速化・障害分離 | 結果整合性・デバッグ複雑化 |
| CDN追加 | 静的資産の高速配信・DDoS耐性 | 動的コンテンツは効果なし・設定ミスリスク |
| シンプルモノリス | 開発速度・デバッグ容易 | スケールしにくい・1箇所の障害が全体に波及 |

**判断基準**: 「今のボトルネックはどこか？」を計測してから追加する。計測なき最適化は過剰設計。

---

## チェックリスト

- [ ] DBへの読み取りリクエストの中で、キャッシュで代替できるものを特定したか
- [ ] APIのレスポンスタイムに影響する重い処理（メール・変換・集計）をキューに移動したか
- [ ] CDNにキャッシュされるリソースに個人データや認証必須コンテンツが含まれていないか
- [ ] 各コンポーネントが落ちたとき（Redisダウン、Pub/Subダウン）にAPIが正しく動作するフォールバックがあるか
- [ ] 構成の変更や選択理由をADR（Architecture Decision Record）として記録したか
