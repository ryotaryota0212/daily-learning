# API設計のベストプラクティス（REST vs GraphQL・バージョニング・レート制限・認証）

## 概要

APIはシステムの「契約」であり、一度公開すると変更コストが高い。設計ミスは技術的負債として長期間残る。
AI時代においても、適切なAPI設計はシステムの疎結合・拡張性・可用性を左右する根幹スキルである。
FastAPI + Cloud Run スタックでは、設計の選択が運用コスト・スケール・セキュリティ全てに直結する。

---

## REST vs GraphQL の判断基準

| 観点 | REST | GraphQL |
|---|---|---|
| データ取得 | エンドポイントごとに固定 | クライアントが必要なフィールドを指定 |
| Over-fetching | 発生しやすい | 発生しにくい |
| キャッシュ | HTTP標準で容易 | 複雑（POSTが多い） |
| 型安全 | OpenAPIで補完 | スキーマが標準で型安全 |
| 複雑な関係データ | N+1問題が起きやすい | 一度に解決できる |
| 学習コスト | 低い | 高い |

**判断のポイント**
- モバイル・SPAで帯域を節約したい → GraphQL
- 単純なCRUD・外部公開API → REST
- 社内ツール・管理画面 → どちらでも良いがREST推奨
- **FastAPI + Cloud Runスタックでは基本REST。複雑なデータグラフが必要になったら追加する**

---

## バージョニング戦略

**3つの方式とトレードオフ**

```
# URLパス（推奨：明示的で分かりやすい）
GET /api/v1/users
GET /api/v2/users

# ヘッダー（URLが綺麗だがクライアント実装が増える）
GET /api/users
Accept: application/vnd.myapp.v2+json

# クエリパラメータ（非推奨：キャッシュ汚染・ログが煩雑）
GET /api/users?version=2
```

**FastAPIでのバージョニング**

```python
from fastapi import APIRouter

v1 = APIRouter(prefix="/api/v1")
v2 = APIRouter(prefix="/api/v2")

@v1.get("/users")
async def get_users_v1(): ...

@v2.get("/users")  # 新形式、v1は維持
async def get_users_v2(): ...

app.include_router(v1)
app.include_router(v2)
```

**廃止ポリシー（必ず決める）**
- 古いバージョンは最低6ヶ月間維持
- `Deprecation` / `Sunset` ヘッダーで事前通知
- ドキュメントに廃止日を明記

---

## アンチパターン vs 正しい設計

**アンチパターン**
```
# NG: 動詞をURLに含める
POST /api/createUser
GET  /api/getUserById?id=1
POST /api/deleteUser

# NG: エラーを常に200で返す
{"status": "error", "message": "Not found"}  # HTTP 200で返す
```

**正しい設計**
```
# OK: リソース名（名詞）+ HTTPメソッドで表現
POST   /api/v1/users        # 作成
GET    /api/v1/users/{id}   # 取得
PATCH  /api/v1/users/{id}   # 部分更新
DELETE /api/v1/users/{id}   # 削除

# OK: HTTPステータスコードを適切に使う
201 Created  → リソース作成成功
400 Bad Request → バリデーションエラー
401 Unauthorized → 認証なし
403 Forbidden → 権限なし
404 Not Found → リソース不存在
429 Too Many Requests → レート制限
```

---

## レート制限の実装パターン

**なぜ必要か**
- DDoS・スクレイピング対策
- 特定ユーザーによるリソース独占防止
- コスト制御（LLM API呼び出し等）

**実装アプローチ（Cloud Run + Redis/Memorystore）**

```python
import time
from fastapi import HTTPException, Request

async def rate_limit(request: Request, user_id: str, limit=100, window=60):
    key = f"rate:{user_id}:{int(time.time() // window)}"
    count = await redis.incr(key)
    if count == 1:
        await redis.expire(key, window)
    if count > limit:
        raise HTTPException(429, detail="Rate limit exceeded")
```

**レスポンスヘッダーで状態を通知**
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1722902400
```

---

## 認証設計のパターン

**Firebase Auth + FastAPIの構成**

```python
from firebase_admin import auth

async def verify_token(authorization: str = Header(...)):
    token = authorization.replace("Bearer ", "")
    try:
        decoded = auth.verify_id_token(token)
        return decoded  # uid, email等が含まれる
    except Exception:
        raise HTTPException(401, detail="Invalid token")
```

**認証 vs 認可を分離する**
- 認証（Authentication）: 誰であるか → Firebase Auth
- 認可（Authorization）: 何ができるか → アプリ側で判断

---

## トレードオフまとめ

| 設計選択 | メリット | デメリット |
|---|---|---|
| URLバージョニング | 明示的・デバッグ容易 | URL設計が複雑化 |
| GraphQL採用 | 柔軟なデータ取得 | キャッシュ・監視が難しい |
| 厳しいレート制限 | リソース保護 | 正規ユーザーへの影響 |
| 下位互換維持 | クライアント影響ゼロ | 保守コスト増大 |

---

## チェックリスト

- [ ] URLはリソース名（名詞）で設計し、動詞はHTTPメソッドで表現している
- [ ] HTTPステータスコードを正しく使い、エラーを200で返していない
- [ ] バージョニング方式と廃止ポリシーを決めてドキュメント化している
- [ ] レート制限を実装し、残り回数をヘッダーで返している
- [ ] 認証（Firebase Auth）と認可（アプリロジック）を分離している
