# システム全体構成の設計パターン（API・DB・キャッシュ・キュー・CDN）

## 概要

「とりあえず動く」から「壊れにくく・スケールする」へ。システム全体の構成要素（API・DB・キャッシュ・メッセージキュー・CDN）をどう組み合わせるかが、AI時代のエンジニアに求められる最重要スキル。コードは生成できても、構成の設計判断は人間が行う。

**なぜ重要か:**
- 構成の選択ミスはリファクタリングでは直せない（アーキテクチャ負債）
- ボトルネックがどこかを事前に把握することで障害を未然に防げる
- 「なぜその構成か」を説明できることがシニアエンジニアの評価基準

---

## 仕組みの要点

### 基本的な構成要素と役割

| 要素 | 役割 | 主な選択肢 |
|------|------|-----------|
| API Gateway | 認証・ルーティング・レート制限 | Cloud Run / FastAPI |
| DB（永続層） | 唯一の真実（Source of Truth） | Neon (PostgreSQL) |
| キャッシュ | 読み取り高速化・DB負荷軽減 | Redis / Memorystore |
| メッセージキュー | 非同期処理・結合度を下げる | Cloud Pub/Sub / Cloud Tasks |
| CDN | 静的コンテンツ配信・エッジ処理 | Cloudflare / Cloud CDN |

### リクエスト処理の典型フロー

```
クライアント
  → CDN（静的リソース）
  → API Gateway（Cloud Run）
      → キャッシュ確認（Redis）→ HIT → 返す
      → キャッシュ MISS → DB（Neon）→ 結果をキャッシュに書く
      → 重い処理 → メッセージキュー → バックグラウンドワーカー
```

### 各コンポーネントが解決する問題

**API (Cloud Run + FastAPI):**
- ステートレス設計で水平スケール可能にする
- JWT検証はAPI層で完結させる（DBに問い合わせない）

**DB (Neon PostgreSQL):**
- すべての永続化はここに集約（複数DBは後から導入）
- Row Level Security（RLS）でテナント分離を強制

**キャッシュ (Redis):**
- 読み取り頻度が高い・更新頻度が低いデータに限定
- キャッシュ戦略はCache-Aside（Lazy Loading）が基本

**メッセージキュー (Cloud Pub/Sub):**
- メール送信・レポート生成など応答不要な処理を分離
- リトライ・デッドレターキューで信頼性を担保

**CDN (Cloudflare):**
- 画像・JS・CSS等の静的ファイル配信
- WAF・DDoS防御の最前線

---

## アンチパターン vs 正しい設計

### アンチパターン

- **DBを直接CDNの裏に置く:** DBコネクションが枯渇する
- **全エンドポイントをキャッシュする:** 更新系APIをキャッシュすると古いデータを返す
- **同期処理で全部やる:** メール送信等で応答遅延、タイムアウトリスク増大
- **最初からマイクロサービス:** 複雑さが爆発、モノリスから始めて分割する

### 正しい設計

- **読み取り集中エンドポイントだけキャッシュ:** GET /products など
- **重い処理はキューに投げてすぐ202を返す:** ユーザー体験を損なわない
- **DBは1つ、アクセス層を分離:** API → Repository層 → DB
- **CDNはAPIではなくアセットに使う:** `/api/*` はオリジン直接

---

## 設計例（FastAPI + Neon + Cloud Run）

```python
# キャッシュ付きの典型エンドポイント
@app.get("/products/{product_id}")
async def get_product(product_id: str, cache: Redis = Depends(get_cache)):
    cached = await cache.get(f"product:{product_id}")
    if cached:
        return json.loads(cached)
    
    product = await db.fetchrow(
        "SELECT * FROM products WHERE id = $1", product_id
    )
    await cache.setex(f"product:{product_id}", 300, json.dumps(product))
    return product

# 非同期タスクの投げ方（Cloud Pub/Sub）
@app.post("/orders")
async def create_order(order: OrderSchema):
    order_id = await db.execute("INSERT INTO orders ...")
    await pubsub_client.publish("order-created", {"order_id": order_id})
    return {"order_id": order_id, "status": "processing"}  # 即返す
```

---

## トレードオフ

| 選択 | メリット | デメリット |
|------|----------|-----------|
| キャッシュ追加 | DB負荷軽減・高速化 | 一貫性の管理が必要 |
| キュー追加 | 非同期・疎結合 | デバッグ難易度が上がる |
| CDN経由API | エッジで処理できる | キャッシュ設定ミスでバグ |
| 単一DB | シンプル・一貫性保証 | スケール限界がある |

**判断の基準:**
- トラフィックが少ない段階では「シンプルな構成 > フル装備」
- ボトルネックを計測してから最適化する（推測しない）
- 各コンポーネントが壊れたときの挙動を必ず設計する

---

## チェックリスト

- [ ] DBへの全アクセスはRepository層を経由しているか
- [ ] キャッシュするのは読み取り専用エンドポイントのみか
- [ ] 重い処理（メール・レポート）はキューで非同期化されているか
- [ ] 各コンポーネントが落ちたときの動作（フォールバック）を定義したか
- [ ] CDNはAPIではなく静的アセットにのみ適用されているか
