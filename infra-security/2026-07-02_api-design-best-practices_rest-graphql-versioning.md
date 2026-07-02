# API設計のベストプラクティス（REST vs GraphQL・バージョニング・レート制限・認証）

## 概要

APIはシステムの「契約」であり、一度公開すると変更コストが高い。
AI時代においては「動くAPIを書く」より「壊れにくく、スケールし、変更しやすいAPIを設計する」力が問われる。
レート制限・バージョニング・認証の設計ミスは、後から直すのが困難なため初期設計が重要。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 適したユースケース | シンプルなCRUD、公開API | 複雑なデータ取得、モバイル最適化 |
| over-fetch / under-fetch | 起きやすい | 起きにくい |
| キャッシュ | HTTPキャッシュが使いやすい | クライアント側での実装が必要 |
| 学習コスト | 低い | 高い |
| N+1問題 | 設計次第 | DataLoaderが必須 |

**FastAPI + Neonスタックでは基本RESTを選ぶ**。GraphQLは複雑なリレーション取得が多い場合のみ検討。

### バージョニング戦略

- **URLパス方式**（推奨）: `/api/v1/users` → 最も明示的
- **ヘッダー方式**: `API-Version: 2024-01` → URLをきれいに保てるが発見しにくい
- **クエリパラメータ方式**: `/users?version=1` → 非推奨（キャッシュ汚染リスク）

### レート制限の設計

- 単位: ユーザーID > IPアドレス（認証済みAPIの場合）
- アルゴリズム: **Token Bucket**（バースト許可）か **Sliding Window**（均等分散）
- ヘッダーで状態を返す: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`
- Cloud Runでは複数インスタンスがあるためRedisで共有カウンター管理が必要

### 認証の設計

- エンドポイントごとに認証要否を明示する（デフォルト認証必須）
- Firebase AuthのJWTをAuthorizationヘッダーで受け取りVerify
- サービス間通信はCloud Run Service Accountで認証（JWTではなくIDトークン）

---

## アンチパターン vs 正しい設計

### アンチパターン

```
# バージョンなし → 将来の変更で既存クライアントが壊れる
GET /users

# エラーレスポンスが不統一 → クライアント実装が複雑になる
{"error": "not found"}  # あるエンドポイント
{"message": "User not found", "code": 404}  # 別のエンドポイント

# 認証をオプションにしてしまう → デフォルトで穴が開く
```

### 正しい設計

```python
# FastAPI での統一エラーレスポンス
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

class APIError(Exception):
    def __init__(self, status_code: int, code: str, message: str):
        self.status_code = status_code
        self.code = code
        self.message = message

@app.exception_handler(APIError)
async def api_error_handler(request: Request, exc: APIError):
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": {"code": exc.code, "message": exc.message}}
    )

# 使用例
raise APIError(404, "USER_NOT_FOUND", "指定されたユーザーが存在しません")
```

---

## コード/設計例：レート制限ミドルウェア（Redis使用）

```python
import redis.asyncio as redis
from fastapi import Request, HTTPException

r = redis.Redis(host="localhost", port=6379)

async def rate_limit(request: Request, limit: int = 100, window: int = 60):
    user_id = request.state.user_id  # Firebase Authで設定済み
    key = f"rate:{user_id}:{window}"
    
    count = await r.incr(key)
    if count == 1:
        await r.expire(key, window)
    
    if count > limit:
        raise HTTPException(status_code=429, detail="Rate limit exceeded")
    
    # レスポンスヘッダーに残量を追加（middlewareで付与）
    request.state.rate_limit_remaining = limit - count
```

---

## トレードオフ

| 選択 | メリット | デメリット |
|------|----------|------------|
| URLバージョニング | 明示的・キャッシュしやすい | URLが長くなる |
| Token Bucket | バースト許可でUX良好 | メモリ消費が多い |
| GraphQL | 柔軟なデータ取得 | N+1・キャッシュ・複雑性 |
| 認証デフォルト必須 | セキュリティが高い | 内部APIも設定が必要 |

---

## チェックリスト

- [ ] 全エンドポイントに `/api/v1/` プレフィックスがある
- [ ] エラーレスポンスが `{"error": {"code": str, "message": str}}` で統一されている
- [ ] レート制限がRedisで共有管理されている（Cloud Run複数インスタンス対応）
- [ ] 認証不要エンドポイントが明示的に列挙されている（デフォルト認証必須）
- [ ] レート制限の残量がレスポンスヘッダーで返されている
