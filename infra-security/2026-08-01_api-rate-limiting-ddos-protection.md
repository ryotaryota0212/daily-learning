# APIレート制限・DDoS対策の実装パターン

## 概要

APIが公開された瞬間から、ブルートフォース・スクレイピング・DDoSのリスクが生まれる。レート制限は「壊れにくいシステムを作る」ための基本防衛であり、単なるセキュリティではなくコスト管理・SLO維持の手段でもある。Cloud Run のコールドスタートやNeonのコネクション上限を守るためにも必須の設計知識。

---

## 仕組みの要点

### レート制限アルゴリズム比較

| アルゴリズム | 特徴 | 向いている用途 |
|---|---|---|
| Fixed Window | シンプル・境界でバースト発生 | 内部API |
| Sliding Window | バースト抑制・計算コスト高 | 外部API |
| Token Bucket | バースト許容・実装複雑 | メディアアップロード |
| Leaky Bucket | 一定流量・遅延が増加 | リアルタイム系 |

### 実装レイヤーの選択肢

- **Cloudflare WAF**: エッジで遮断。オリジンに届く前にDDoSを吸収。最も効果的
- **FastAPIミドルウェア**: IPやユーザーID単位の細粒度制御が可能
- **Cloud Run前段のLoad Balancer**: Cloud Armorで地域制限・レート制限

---

## アンチパターン vs 正しい設計

### アンチパターン
- インメモリカウンタで制限 → Pod再起動やスケールアウトで状態がリセット
- IPのみでレート制限 → 正規ユーザーがNATの背後にいる場合に誤爆
- エラーレスポンスに制限情報を返さない → クライアントが適切にリトライできない

### 正しい設計
- **Redisで分散カウンタ管理**（Cloud Memorystore または Upstash）
- **キー設計**: `rate:{user_id}:{endpoint}:{window}` で細粒度に制御
- **Retry-After ヘッダー**を必ず返す
- **X-RateLimit-Limit / X-RateLimit-Remaining / X-RateLimit-Reset** を返す

---

## コード例（FastAPI + Redis）

```python
import time
import redis
from fastapi import Request, HTTPException

r = redis.Redis(host="redis-host", decode_responses=True)

async def rate_limit(request: Request, limit: int = 60, window: int = 60):
    user_id = request.state.user_id  # Firebase Auth で取得済み
    key = f"rate:{user_id}:{int(time.time() // window)}"
    
    count = r.incr(key)
    if count == 1:
        r.expire(key, window)
    
    if count > limit:
        raise HTTPException(
            status_code=429,
            detail="Too Many Requests",
            headers={"Retry-After": str(window)}
        )
```

---

## DDoS対策の多層防御

```
Internet → Cloudflare WAF → Cloud Load Balancer → Cloud Run → FastAPI
              ↓                    ↓                  ↓
          Bot/DDoS遮断        Cloud Armor         ユーザー単位制限
          地域制限IP           IPレート制限         エンドポイント別制限
```

- **Layer 3/4**: Cloudflareが自動吸収（volumetric DDoS）
- **Layer 7**: WAFルールで異常なリクエストパターンを遮断
- **アプリ層**: 認証済みユーザー単位でAPIごとに異なる制限を適用

---

## トレードオフ

| 選択肢 | メリット | デメリット |
|---|---|---|
| Cloudflareのみ | 運用シンプル・コスト低 | ユーザー単位制御不可 |
| アプリ内のみ | 細粒度制御 | オリジンまで到達・Redis依存 |
| 多層防御 | 最強 | 複雑・コスト高 |

**現実解**: Cloudflare で粗い保護 + アプリ内で認証ユーザー単位の細粒度制限。

---

## チェックリスト

- [ ] レート制限のカウンタはRedis等の共有ストアで管理している
- [ ] 429レスポンスに `Retry-After` ヘッダーを含んでいる
- [ ] エンドポイントごとに異なる制限値を設定できる
- [ ] Cloudflare WAF / Cloud Armor でエッジ保護を設定している
- [ ] レート制限超過のログ・アラートを設定し、攻撃を検知できる
