# API設計のベストプラクティス — REST vs GraphQL、バージョニング、レート制限、認証

## 概要

APIはシステムの「契約書」であり、一度公開すると変更コストが非常に高い。
AI時代においてAPIの重要性はむしろ増している：LLMや外部サービスとの統合が前提となるからだ。
「とりあえず動くAPI」を作るのではなく、「何年後も壊れにくく、使いやすいAPI」を設計する視点が求められる。
設計の誤りはクライアント側の大規模修正や、セキュリティ事故に直結する。

---

## 仕組みの要点

### REST vs GraphQL — 選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| ユースケース | シンプルなCRUD、外部公開API | 複雑なリレーション、フロントエンド主導 |
| 学習コスト | 低い | 高い（スキーマ定義必要） |
| キャッシュ | CDN・HTTPキャッシュが素直に使える | 複雑（POSTベース） |
| 型安全 | OpenAPIで補完 | スキーマが型の源泉 |
| オーバーフェッチ | 起きやすい | 回避しやすい |

**判断基準：**
- 外部パートナーや一般公開 → REST（理解しやすく、ツールが豊富）
- 自社フロントエンド専用、データ関係が複雑 → GraphQL検討
- FastAPI + Cloud Run 構成の場合は REST が初速が高く運用負荷が低い

### バージョニング戦略

- **URLパス方式**（推奨）: `/v1/users`, `/v2/users` — 明示的でキャッシュと相性が良い
- **ヘッダー方式**: `API-Version: 2` — URLを汚さないが、デバッグしにくい
- **クエリパラメータ方式**: `?version=2` — ログ解析が楽だが混在しやすい

**方針：**
- 破壊的変更（フィールド削除、型変更）は必ずメジャーバージョンを上げる
- 旧バージョンは最低6ヶ月は並行運用し、廃止予告ヘッダー（`Sunset`）を返す
- 非破壊的変更（フィールド追加）はバージョンを上げない

### レート制限の設計

- **単位**: ユーザーID単位 > IP単位（NAT環境で共有IPは使えない）
- **アルゴリズム**:
  - Token Bucket: バースト許容（推奨）
  - Fixed Window: シンプルだが境界でスパイク発生
  - Sliding Window: 精度が高いがRedis負荷高
- **レスポンスヘッダー**でクライアントに残量を伝える:
  ```
  X-RateLimit-Limit: 1000
  X-RateLimit-Remaining: 750
  X-RateLimit-Reset: 1717344000
  ```
- Cloud Run 前段に Cloud Armor や Cloudflare を置いてエッジでも制限する

---

## アンチパターン vs 正しい設計

### アンチパターン

```
# ❌ 動詞をURLに入れる
POST /getUser
POST /createUser
POST /deleteUser

# ❌ エラーを全部200で返す
{"status": "error", "message": "Not found"}  # HTTP 200

# ❌ バージョンなしで破壊的変更
DELETE /users/{id}/name  # 突然フィールドを削除

# ❌ 全員に同じレート制限
# 無料ユーザーも有料ユーザーも同じ1000 req/hで良いか？
```

### 正しい設計

```
# ✅ リソース指向のRESTful設計
GET    /v1/users/{id}        # 取得
POST   /v1/users             # 作成
PATCH  /v1/users/{id}        # 部分更新
DELETE /v1/users/{id}        # 削除

# ✅ HTTPステータスコードを正しく使う
200 OK / 201 Created / 204 No Content
400 Bad Request / 401 Unauthorized / 403 Forbidden / 404 Not Found
429 Too Many Requests / 500 Internal Server Error

# ✅ 廃止予告ヘッダー
Sunset: Sat, 01 Jan 2027 00:00:00 GMT
Deprecation: true
```

---

## コード / 設計例（FastAPI）

```python
from fastapi import FastAPI, Depends, HTTPException, Request
from fastapi.responses import JSONResponse

app = FastAPI()

# バージョン管理 - ルーターで分離
from fastapi import APIRouter
v1_router = APIRouter(prefix="/v1")
v2_router = APIRouter(prefix="/v2")

# レート制限ミドルウェア（Redis使用）
async def rate_limit(request: Request, user_id: str):
    key = f"rate:{user_id}"
    count = await redis.incr(key)
    if count == 1:
        await redis.expire(key, 3600)  # 1時間ウィンドウ
    if count > 1000:
        raise HTTPException(
            status_code=429,
            headers={"Retry-After": "3600"},
            detail="Rate limit exceeded"
        )

# 認証 - Firebase Auth トークン検証
async def verify_token(request: Request) -> dict:
    token = request.headers.get("Authorization", "").replace("Bearer ", "")
    try:
        decoded = auth.verify_id_token(token)
        return decoded
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")

@v1_router.get("/users/{user_id}")
async def get_user(user_id: str, user=Depends(verify_token)):
    # 自分のリソースか確認（403 vs 404の使い分け）
    if user["uid"] != user_id:
        raise HTTPException(status_code=403, detail="Forbidden")
    return {"id": user_id, "email": user["email"]}
```

---

## トレードオフ

| 判断 | メリット | デメリット |
|------|----------|------------|
| RESTを選ぶ | 学習コスト低・キャッシュ容易 | オーバーフェッチが起きやすい |
| GraphQLを選ぶ | 柔軟なクエリ・型安全 | N+1問題・キャッシュ複雑 |
| URLバージョニング | 明示的・シンプル | URLが長くなる |
| 厳しいレート制限 | DoS耐性高い | 正当なユーザーを弾く可能性 |
| 認証を全エンドポイントに | セキュア | 公開APIとの設計が複雑化 |

**実際の選択指針:**
- FastAPI + Cloud Run 構成では REST + URLバージョニングが最も素直
- レート制限は Cloud Armor（エッジ）+ アプリ層（ユーザー単位）の二層で実装
- 403と404の使い分け: リソースの存在を漏らしたくなければ全て404を返す

---

## チェックリスト

- [ ] URLはリソース指向（動詞でなく名詞）で設計されているか
- [ ] 破壊的変更時はバージョンを上げ、旧バージョンに廃止予告ヘッダーを返しているか
- [ ] レート制限はIP単位ではなく認証済みユーザー単位で行っているか
- [ ] 401（未認証）と403（認可不足）を正しく使い分けているか
- [ ] エラーレスポンスは一貫したフォーマット（`detail`フィールド等）になっているか
