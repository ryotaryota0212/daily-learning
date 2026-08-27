# システム全体構成設計パターン（API・DB・キャッシュ・キュー・CDN）

## 概要

AI時代に「コードを書く力」より「システムを成立させる力」が重要になっている。  
適切なコンポーネント選択と配置で、スケーラビリティ・可用性・保守性を両立させる。  
構成要素の役割と連携を理解し、トレードオフを説明できる設計者になることが本質。  
「とりあえず動く」ではなく「壊れにくく、スケールしやすい」構成を選べるかが問われる。

## 仕組みの要点

### 基本コンポーネントの役割

| コンポーネント | 役割 | 選定例 |
|---|---|---|
| API層 | リクエスト処理・認証・ビジネスロジック | Cloud Run (FastAPI) |
| DB | 永続化・ACID保証・複雑クエリ | Neon (PostgreSQL) |
| キャッシュ | 高頻度読み取り・セッション・レート制限 | Upstash Redis |
| キュー | 非同期処理・ピーク負荷の吸収・リトライ | Cloud Pub/Sub |
| CDN | 静的配信・エッジキャッシュ・DDoS防御 | Cloudflare |

### 設計の3原則

- **単一責任**: 各コンポーネントは1つの役割に集中させる
- **疎結合**: コンポーネント間はAPI/イベントで接続し、直接依存を避ける
- **データ局所性**: 処理の近くにデータを置く（エッジキャッシュ、レプリカ）

## アンチパターン vs 正しい設計

| アンチパターン | 問題 | 正しい設計 |
|---|---|---|
| DBに全アクセス集中 | 高負荷でDBがボトルネック | キャッシュで読み取りをオフロード |
| 重い処理を同期API内で実行 | タイムアウト・UX悪化 | キューで非同期化 |
| APIがDB操作とビジネスロジック混在 | テスト不能・拡張困難 | レイヤー分離（Router→Service→Repository） |
| 単一リージョン・単一インスタンス | 単一障害点（SPOF） | CDN + オートスケール + フェイルオーバー |

## 構成図と実装例

```
[User] → [Cloudflare CDN] → [Cloud Run: API (FastAPI)]
                                    ↓              ↓
                            [Upstash Redis]   [Neon PostgreSQL]
                                    ↓
                            [Cloud Pub/Sub]
                                    ↓
                            [Cloud Run: Worker]
```

```python
# Cache-Aside パターン（最も基本的なキャッシュ戦略）
async def get_user(user_id: str):
    cached = await redis.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)  # キャッシュヒット

    user = await db.fetchrow(
        "SELECT * FROM users WHERE id=$1", user_id
    )
    if user:
        await redis.setex(f"user:{user_id}", 300, json.dumps(dict(user)))
    return user
```

```python
# 重い処理をキューに委譲
async def create_report(user_id: str, params: dict):
    message = {"user_id": user_id, "params": params}
    await pubsub_client.publish(TOPIC, json.dumps(message).encode())
    return {"status": "queued"}  # 即時レスポンス
```

## トレードオフ

| 選択 | メリット | デメリット |
|---|---|---|
| キャッシュ追加 | レイテンシ改善、DB負荷軽減 | 整合性管理が複雑化 |
| キュー導入 | ピーク耐性・非同期化 | 最終整合性・デバッグ難化 |
| CDN | グローバル配信速度向上 | キャッシュパージの管理 |
| マイクロサービス化 | スケール独立性 | 運用・分散トレーシングの複雑化 |

**推奨**: スタートはモノリス+キャッシュ+CDN。ボトルネックが明確になったらキュー導入→スケール分離の順で進化させる。

## チェックリスト

- [ ] 各コンポーネントのボトルネック（CPU/メモリ/IO）を特定できるか
- [ ] DBへの直接アクセスをキャッシュでオフロードしているか
- [ ] 重い処理は非同期キューで処理しているか（同期API内で実行していないか）
- [ ] 単一障害点（SPOF）がなく、1コンポーネント障害時の影響範囲を説明できるか
- [ ] コスト・レイテンシ・運用複雑度のバランスをトレードオフとして説明できるか
