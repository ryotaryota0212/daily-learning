# キャッシュ戦略の設計 — Cache-Aside, Write-Through, TTL設計, キャッシュ無効化

## 概要

キャッシュはシステムの応答速度とスケーラビリティを劇的に改善する最も費用対効果の高い手段。
しかし「とりあえずキャッシュを入れる」は障害の温床になる。
**何を・どこで・いつまで・どう無効化するか**を設計できることが、AI時代のシステム設計者に求められる力。
キャッシュの設計ミスはデータ不整合・メモリ枯渇・thundering herd問題として表面化する。

## キャッシュパターンの要点

### Cache-Aside（Lazy Loading）
- アプリがキャッシュを確認 → ミス時にDBから取得 → キャッシュに書き込み
- **最も一般的**。読み取り頻度が高く、更新頻度が低いデータ向き
- 欠点: 初回アクセスは必ずキャッシュミス（cold start問題）

### Write-Through
- 書き込み時にDB + キャッシュの両方を同時更新
- データの一貫性が高い。書き込みレイテンシは増加
- 読まれないデータもキャッシュされる無駄がある

### Write-Behind（Write-Back）
- キャッシュに書き込み → 非同期でDBに反映
- 書き込みレイテンシは最小。**データ消失リスクあり**
- Cloud Run のようなステートレス環境では非推奨

### Read-Through
- キャッシュ側がDB取得を担当（アプリはキャッシュのみ参照）
- Cache-Asideとの違いはロジックの置き場所

## アンチパターン vs 正しい設計

| アンチパターン | 問題点 | 正しい設計 |
|---|---|---|
| 全レスポンスを一律キャッシュ | メモリ枯渇、古いデータ表示 | ホットデータのみ選別してキャッシュ |
| TTLなしでキャッシュ | データが永遠に古いまま | 用途に応じたTTL設定（後述） |
| キャッシュ障害時にシステム停止 | SPOFになる | キャッシュミス時はDBフォールバック |
| 更新時にキャッシュを「更新」 | レースコンディション | 更新時はキャッシュを**削除**（次読み取りで再構築） |
| 全ユーザーで同じTTL | thundering herd発生 | TTLにジッター（±ランダム秒）を付与 |

## TTL設計の指針

| データ種別 | TTL目安 | 理由 |
|---|---|---|
| マスタデータ（国、カテゴリ） | 1時間〜1日 | 変更頻度が極めて低い |
| ユーザープロフィール | 5〜15分 | 更新はあるが即時反映不要 |
| 認証トークン検証結果 | 1〜5分 | セキュリティとのバランス |
| APIレスポンス（リスト系） | 30秒〜5分 | 鮮度とDB負荷のトレードオフ |
| リアルタイムデータ（在庫等） | キャッシュしない | 不整合の影響が大きい |

## コード例（FastAPI + Redis / Cache-Aside）

```python
from fastapi import FastAPI, Depends
import redis.asyncio as redis
import json

app = FastAPI()
cache = redis.from_url("redis://localhost:6379")

async def get_user(user_id: str):
    key = f"user:{user_id}"
    cached = await cache.get(key)
    if cached:
        return json.loads(cached)
    # キャッシュミス → DB取得
    user = await db_fetch_user(user_id)
    if user:
        jitter = random.randint(-30, 30)
        await cache.set(key, json.dumps(user), ex=300 + jitter)
    return user

async def update_user(user_id: str, data: dict):
    await db_update_user(user_id, data)
    await cache.delete(f"user:{user_id}")  # 更新ではなく削除
```

## キャッシュ無効化の戦略

- **TTL期限切れ**: 最もシンプル。許容可能な遅延がある場合に最適
- **イベント駆動削除**: DB更新時にキャッシュキーを明示的に削除
- **バージョンキー**: `user:123:v5` のようにバージョンを含め、更新時にバージョンを上げる
- **Pub/Sub通知**: Cloud Pub/Sub で複数インスタンス間のキャッシュ同期

## トレードオフ

| 観点 | キャッシュあり | キャッシュなし |
|---|---|---|
| レイテンシ | 低い（1-5ms） | DB依存（10-100ms） |
| データ鮮度 | TTL分の遅延あり | 常に最新 |
| 運用複雑性 | 高い（Redis管理、無効化ロジック） | 低い |
| コスト | Redis費用が追加 | DB負荷増 → スケール費用増 |
| 障害パターン | キャッシュ不整合、thundering herd | DBボトルネック |

### Cloud Run環境での注意点
- Cloud Runはインスタンスが動的に増減 → **インメモリキャッシュは共有不可**
- Memorystore（Redis）またはCloud CDNを外部キャッシュとして使用
- インメモリは「TTL数秒のL1キャッシュ」としてのみ活用

## チェックリスト

- [ ] キャッシュ対象データの選定基準（読み取り頻度/更新頻度）を明文化したか
- [ ] TTLはデータ種別ごとに設定し、ジッターを付与しているか
- [ ] キャッシュ障害時のフォールバック（DB直接アクセス）が実装されているか
- [ ] データ更新時のキャッシュ無効化戦略が「更新」ではなく「削除」になっているか
- [ ] thundering herd対策（ジッター、ロック、stale-while-revalidate）を検討したか
