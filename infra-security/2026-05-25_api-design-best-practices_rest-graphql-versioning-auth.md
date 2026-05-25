# API設計のベストプラクティス：REST vs GraphQL、バージョニング、レート制限、認証

## 概要

API設計はシステムの「契約」であり、一度公開すると変更コストが極めて高い。  
AI時代においても「どのAPIを切るか・どう守るか」はシステム全体の品質を左右する判断軸であり、  
コードを書く前に設計の正しさを問う姿勢が求められる。  
FastAPI + Cloud Run + Firebase Auth スタックでは、設計の誤りがスケール時に深刻な負債になる。

---

## 仕組みの要点

### REST vs GraphQL の選び方

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いているケース | リソースが明確、外部公開API | UIドリブン、フロント要件が変わりやすい |
| N+1問題 | URLごとに管理しやすい | DataLoaderが必要 |
| キャッシュ | CDNキャッシュがしやすい | POSTが多くCDN不向き |
| 型安全性 | OpenAPI(Swagger)で担保 | スキーマが強力 |

- **モバイル・外部パートナー向け → REST**（変更しにくい契約が明確になる）
- **社内BFF・フロント専用 → GraphQL**（UIの柔軟性が活きる）
- FastAPIスタックでは **REST + OpenAPI自動生成** がデフォルト最適解

### バージョニング戦略

- **URLパス方式**（推奨）: `/v1/users`, `/v2/users`
  - メリット: CDNキャッシュしやすい、ルーティングが明確
- **ヘッダー方式**: `Accept: application/vnd.api+json;version=2`
  - デメリット: テストが面倒、ブラウザからのアクセスが難しい
- **クエリパラメータ方式**: `?version=2` → 非推奨（ログが汚れる）

**ルール**: v1を廃止する前に最低6ヶ月の移行期間を設ける。`Sunset` ヘッダーで廃止日を通知。

---

## アンチパターン vs 正しい設計

### アンチパターン

```
# NG: 動詞をエンドポイントに含める
POST /getUser
POST /createUserAndSendEmail  # 複数責務

# NG: バージョン管理なしで破壊的変更
# フィールド名を変更 → 既存クライアントが全滅

# NG: 認証なしの内部エンドポイント
GET /internal/admin/users  # ネットワーク境界だけに頼る
```

### 正しい設計

```
# OK: リソース名詞 + HTTPメソッドで意図を表現
GET    /v1/users/{id}       # 取得
POST   /v1/users            # 作成
PATCH  /v1/users/{id}       # 部分更新
DELETE /v1/users/{id}       # 削除

# OK: 副作用を伴うアクションはサブリソースで表現
POST /v1/users/{id}/activate
POST /v1/orders/{id}/cancel
```

---

## FastAPI実装例（認証 + レート制限）

```python
from fastapi import FastAPI, Depends, HTTPException, Request
from firebase_admin import auth
import time
from collections import defaultdict

app = FastAPI()
_rate_store: dict[str, list[float]] = defaultdict(list)

def rate_limit(request: Request, limit: int = 60, window: int = 60):
    key = request.client.host
    now = time.time()
    hits = [t for t in _rate_store[key] if now - t < window]
    if len(hits) >= limit:
        raise HTTPException(status_code=429, detail="Rate limit exceeded")
    _rate_store[key] = hits + [now]

async def verify_token(request: Request) -> dict:
    token = request.headers.get("Authorization", "").removeprefix("Bearer ")
    try:
        return auth.verify_id_token(token)
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")

@app.get("/v1/users/me")
async def get_me(
    _: None = Depends(rate_limit),
    user: dict = Depends(verify_token),
):
    return {"uid": user["uid"], "email": user.get("email")}
```

> 本番では `_rate_store` を Redis に置き換える（Cloud Run は複数インスタンスのため）。

---

## レート制限の設計パターン

| アルゴリズム | 特徴 | 向いているケース |
|------------|------|----------------|
| Fixed Window | シンプル、境界でバースト | 社内API |
| Sliding Window | バースト防止、精度高い | 外部公開API |
| Token Bucket | バーストを許容しつつ平均を制御 | ストリーミング系 |

- Cloud Run + Redis なら **Sliding Window** を推奨
- レート制限のキーは **IPではなくUID**（同一ユーザーの複数デバイス対策）
- 超過時は `Retry-After` ヘッダーをレスポンスに含める

---

## トレードオフ

| 選択肢 | メリット | デメリット |
|--------|---------|-----------|
| REST | シンプル・キャッシュしやすい | 複雑なクエリで over-fetch/under-fetch |
| GraphQL | フロント自由度が高い | N+1・キャッシュ設計が難しい |
| URLバージョニング | 運用しやすい | URL が増える |
| 認証をAPIGWに委譲 | 一元管理 | APIGW障害が全体に波及 |
| アプリ内認証 | 柔軟 | 各サービスで実装が分散 |

---

## チェックリスト

- [ ] エンドポイントはリソース名詞 + HTTPメソッドで設計されているか
- [ ] バージョンはURLパスで管理し、廃止スケジュールが決まっているか
- [ ] レート制限はUIDベースで実装し、Redisに状態を持っているか
- [ ] 全エンドポイントにFirebase Auth検証が適用されているか（内部エンドポイント含む）
- [ ] 429 / 401 レスポンスに `Retry-After` / `WWW-Authenticate` ヘッダーがあるか
