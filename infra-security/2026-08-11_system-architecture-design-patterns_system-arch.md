# システム全体構成の設計パターン（API・DB・キャッシュ・キュー・CDN）

## 概要

AI時代において「コードを書く力」より「システムを成立させる力」が問われる。
その核心が「どのコンポーネントをどう組み合わせるか」の設計判断だ。
DBに全部詰め込む設計、キャッシュなしで毎回DBを叩く設計、同期処理で全部繋ぐ設計——
これらは動くが壊れやすく、スケールしない。正しい構成パターンを知ることが設計力の出発点。

## 仕組みの要点

### 主要コンポーネントの役割

| コンポーネント | 役割 | 使い所 |
|---|---|---|
| API (FastAPI) | ロジック・バリデーション・認証 | リクエスト処理のハブ |
| DB (Neon/PostgreSQL) | 永続化・整合性 | 真実の源泉（Source of Truth） |
| Cache (Redis) | 高速読み取り・一時データ | 読み多・書き少のデータ |
| Queue (Pub/Sub等) | 非同期・デカップリング | 重い処理・外部連携 |
| CDN (Cloudflare) | 静的配信・エッジキャッシュ | 静的アセット・API加速 |

### リクエストフローの基本パターン

```
User → CDN → API Gateway → FastAPI → Cache → DB
                                  ↓
                               Queue → Worker → 外部API/通知
```

- **CDN**: 静的ファイルはオリジンに届かせない
- **Cache**: 読み取りはまずRedisへ、ミスしたらDBへ（Cache-Aside）
- **Queue**: 応答に不要な処理（メール送信・集計）はすべてキューへ

### 各層の設計判断基準

- **APIサーバー（Cloud Run）**: ステートレスで水平スケール前提。セッション状態をサーバーに持たない
- **DB**: 書き込みの正規化・インデックス・RLSで整合性を守る。スケールは後から
- **Cache**: TTLを短く設定し、キャッシュ破損より古いデータのほうがマシ
- **Queue**: ワーカーは必ずべき等（同じメッセージを2回処理しても壊れない）に設計

## アンチパターン vs 正しい設計

| アンチパターン | 問題 | 正しい設計 |
|---|---|---|
| 毎リクエストでDB全件取得 | DB過負荷 | ページネーション＋キャッシュ |
| APIサーバーにセッション保存 | スケールアウトできない | JWTまたはRedisセッション |
| 外部API呼び出しをリクエスト内で同期実行 | タイムアウトリスク | Queueに積んで非同期処理 |
| 全テーブルを結合して取得 | クエリが遅い | 非正規化 or ビュー or Cache |
| エラー時にリトライなし | メッセージ消失 | Dead Letter Queue（DLQ）設計 |

## 設計例：ユーザー投稿フィードAPI（FastAPI + Neon + Redis）

```python
# Cache-Asideパターン + 非同期通知のフロー
@app.get("/feed")
async def get_feed(user_id: str, redis: Redis, db: AsyncSession):
    cache_key = f"feed:{user_id}"
    cached = await redis.get(cache_key)
    if cached:
        return json.loads(cached)

    posts = await db.execute(
        select(Post).where(Post.user_id == user_id).limit(20)
    )
    result = [p.to_dict() for p in posts.scalars()]
    await redis.setex(cache_key, 60, json.dumps(result))  # TTL 60秒
    return result

@app.post("/posts")
async def create_post(data: PostCreate, pubsub: PubSubClient):
    post = await save_to_db(data)
    # 通知・集計はキューへ（レスポンスをブロックしない）
    await pubsub.publish("post.created", {"post_id": post.id})
    return {"id": post.id}
```

## トレードオフ

| 選択 | メリット | デメリット |
|---|---|---|
| キャッシュ積極利用 | 高速・DB負荷減 | データの鮮度低下・無効化が複雑 |
| 非同期Queue多用 | 疎結合・耐障害性 | デバッグ難・整合性の遅延 |
| CDNキャッシュ | グローバル高速化 | Purgeタイミングの管理必要 |
| モノリシックDB | 整合性維持が容易 | スケールの上限あり |

### Cloud Run + Neon + Firebase Auth スタックでの注意点

- Cloud RunはスケールアウトするためDBコネクションプールに注意（PgBouncerや`pool_mode=transaction`推奨）
- Firebase AuthのJWT検証はAPIサーバー側で毎回行わず、結果を短時間キャッシュ
- Neonはサーバーレスでコールドスタートあり→`pool_timeout`を適切に設定

## チェックリスト

- [ ] 読み取りの多いエンドポイントにキャッシュ層を設けているか
- [ ] APIサーバーがステートレスか（セッション・状態をサーバーに持っていないか）
- [ ] 重い処理・外部API呼び出しをQueueに切り出しているか
- [ ] DBコネクションプールを設定しているか（Cloud Run × Neonの場合は必須）
- [ ] キューのワーカーがべき等に設計されているか（再処理しても壊れないか）
