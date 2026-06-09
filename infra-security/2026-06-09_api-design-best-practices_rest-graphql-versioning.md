# API設計のベストプラクティス：REST vs GraphQL・バージョニング・レート制限・認証

## 概要

API設計はシステムの「契約」である。一度公開すると変更が難しく、設計ミスはクライアント全体に波及する。
AI時代においても「APIをどう切るか」「どう守るか」「どう進化させるか」を判断できる力は不可欠。
単に動くエンドポイントを作るのではなく、スケール・保守性・セキュリティを同時に満たす設計が求められる。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 適したユースケース | リソース単位のCRUD、公開API | 複雑な関連データ取得、BFF |
| Over-fetching/Under-fetching | 起きやすい | クライアントが必要なフィールドのみ取得可能 |
| キャッシュ | URLベースで容易（CDN対応しやすい） | クエリが複雑でキャッシュが難しい |
| セキュリティ | エンドポイント単位で制御しやすい | クエリの深さ・複雑度制限が必要 |
| 学習コスト | 低い | 高い（スキーマ・リゾルバーの設計が必要） |

**判断基準：** 外部公開APIや単純なCRUDはREST。モバイルBFFや複雑なデータグラフはGraphQL。迷ったらREST。

### バージョニング戦略

- **URLパス方式**（推奨）: `/api/v1/users` — 明確で CDN・ログ・デバッグが簡単
- **ヘッダー方式**: `Accept: application/vnd.api+json;version=2` — URLが綺麗だが運用が複雑
- **クエリパラメータ方式**: `/users?version=1` — キャッシュが壊れやすいため非推奨

**バージョン管理ルール：**
- 後方互換のある変更（フィールド追加）はバージョン不要
- 破壊的変更（フィールド削除・型変更・URL変更）は必ずメジャーバージョンを上げる
- 旧バージョンは最低6ヶ月はサポートし、廃止予告ヘッダー (`Sunset: Sat, 01 Jan 2027 00:00:00 GMT`) を返す

### レスポンス設計

- 成功: `200 OK`（取得）、`201 Created`（作成）、`204 No Content`（削除）
- クライアントエラー: `400`（バリデーション）、`401`（未認証）、`403`（権限不足）、`404`（未存在）、`409`（競合）、`422`（処理不能エンティティ）
- サーバーエラー: `500`（内部エラー）、`503`（サービス不能）
- エラーレスポンスは必ず構造化する: `{"error": {"code": "USER_NOT_FOUND", "message": "..."} }`

---

## アンチパターン vs 正しい設計

### アンチパターン

```
# NG: 動詞をURLに含める
POST /api/getUsers
POST /api/deleteUser?id=1
GET  /api/createOrder

# NG: エラーを常に200で返す
HTTP/1.1 200 OK
{"success": false, "error": "not found"}

# NG: バージョニングなしで破壊的変更
DELETE /api/users/:id のレスポンスを突然変更
```

### 正しい設計

```
# OK: リソース + HTTPメソッドで表現
GET    /api/v1/users          # 一覧
POST   /api/v1/users          # 作成
GET    /api/v1/users/:id      # 取得
PATCH  /api/v1/users/:id      # 部分更新
DELETE /api/v1/users/:id      # 削除

# OK: 関連リソースのネスト（深さは2階層まで）
GET /api/v1/users/:id/orders
```

---

## コード例：FastAPI での実装パターン

```python
from fastapi import APIRouter, Depends, HTTPException, Request
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
router = APIRouter(prefix="/api/v1", tags=["users"])

@router.get("/users/{user_id}")
@limiter.limit("60/minute")  # レート制限
async def get_user(
    request: Request,
    user_id: str,
    current_user = Depends(verify_firebase_token),  # Firebase Auth
):
    if current_user.uid != user_id and not current_user.is_admin:
        raise HTTPException(status_code=403, detail="Forbidden")
    user = await db.fetch_user(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user

# エラーハンドラー（構造化エラー）
@app.exception_handler(HTTPException)
async def http_exception_handler(request, exc):
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": {"code": exc.detail, "status": exc.status_code}}
    )
```

---

## レート制限設計

- **目的**: DDoS軽減、コスト制御、公平な利用保証
- **粒度**: IP単位 < ユーザー単位 < APIキー単位（精度が上がるほど実装が複雑）
- **アルゴリズム**:
  - Token Bucket: バーストを許容しつつ平均レートを制限（推奨）
  - Fixed Window: 実装が簡単だが境界攻撃に弱い
  - Sliding Window: 精度が高いがRedisのメモリを消費

**ヘッダーで制限状態を返す:**
```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1749470400
Retry-After: 30  # 429時のみ
```

---

## 認証設計

- **Firebase Auth + JWT**: Cloud Run環境での推奨。`Authorization: Bearer <token>` で送信
- **検証**: JWTの署名をFirebase公開鍵で検証、`exp`/`aud`/`iss` を必ず確認
- **サービス間認証**: Cloud Run同士はGoogle-managed Identity Token を使用（IAM バインディング）

```python
async def verify_firebase_token(authorization: str = Header(...)):
    token = authorization.replace("Bearer ", "")
    decoded = firebase_admin.auth.verify_id_token(token)  # 署名・有効期限を自動検証
    return decoded
```

---

## トレードオフ

| 選択 | メリット | デメリット |
|------|---------|---------|
| REST | シンプル・キャッシュ容易・広い知識ベース | over-fetchingが起きやすい |
| GraphQL | 柔軟なクエリ・型安全 | 複雑度管理が必要、CDNキャッシュ困難 |
| URLバージョニング | 明確・運用しやすい | URLが冗長になる |
| レート制限をRedisで実装 | 分散対応・精度が高い | Redis障害がAPIに波及するリスク |

---

## チェックリスト

- [ ] エンドポイントはリソース名（名詞）+ HTTPメソッドで設計されているか
- [ ] 破壊的変更時はバージョンを上げ、旧バージョンに `Sunset` ヘッダーを返しているか
- [ ] エラーレスポンスは構造化され、適切なHTTPステータスコードを返しているか
- [ ] レート制限が実装され、制限状態をヘッダーで返しているか
- [ ] 認証トークンの署名・有効期限・`aud`/`iss` を検証しているか
