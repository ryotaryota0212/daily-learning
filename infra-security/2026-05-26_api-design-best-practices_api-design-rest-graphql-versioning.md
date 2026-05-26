# API設計のベストプラクティス — REST vs GraphQL・バージョニング・レート制限・認証

## 概要

APIはシステムの「契約」。一度公開したAPIは変更コストが高く、設計ミスが後から波及する。
AI時代でも「どういうAPIを設計するか」の意思決定はエンジニアが行う必要がある。
FastAPI + Cloud Run 構成では、適切な設計が可用性・セキュリティ・スケーラビリティすべてに影響する。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いているケース | シンプルなCRUD、公開API | 複雑な関連データ、BFF層 |
| N+1問題 | エンドポイント設計で回避 | DataLoader必須 |
| キャッシュ | HTTP標準キャッシュが使いやすい | クエリごとにキャッシュ難しい |
| 学習コスト | 低 | 高 |
| FastAPI相性 | ◎ | △（Strawberryで可能） |

**結論**: 社内API・モバイルBFF以外は REST を選ぶのが無難。GraphQLは「複数クライアントが異なる形でデータを取る」ときのみ検討。

### バージョニング戦略

- **URLパス方式** (`/v1/users`) — 最も明確。推奨
- **ヘッダー方式** (`API-Version: 2024-01`) — URLを汚さない。Stripe方式
- **クエリパラメータ方式** (`?version=1`) — 非推奨。キャッシュと相性が悪い

**原則**:
- v1 は壊さない。後方互換を維持する
- 破壊的変更は新バージョンに切る
- 旧バージョンのサポート終了日を明示する（Sunset ヘッダー）

### レート制限の設計

- 単位: ユーザーIDまたはAPIキーごと（IPだけでは不十分）
- アルゴリズム: **Sliding Window** が最も公平（Fixed WindowはバーストNG）
- ヘッダーで状態を返す: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`
- 超過時: `429 Too Many Requests` + `Retry-After` ヘッダー

### 認証パターン

- **Firebase Auth JWT** → `Authorization: Bearer <token>` → Cloud Runで検証
- サービス間通信 → Google サービスアカウント + ID Token（Firebase JWTと混同しない）
- APIキー → 内部ツール・Webhook向け。JWTより単純だが失効管理が必要

---

## アンチパターン vs 正しい設計

### アンチパターン

```
# 悪い例: 動詞URLとエラーレスポンスが統一されていない
GET /getUser?userId=123
POST /deleteUser
# エラーレスポンスが {"error": "not found"} だったり {"message": "error"} だったり
```

- すべてのエラーが `200 OK` で返る
- エラーレスポンスのスキーマが統一されていない
- ページネーションがない（全件返す）
- `id` を連番整数で公開（列挙攻撃リスク）

### 正しい設計

```
# 良い例: リソース指向 + 統一エラースキーマ
GET    /v1/users/{user_id}   → 200 or 404
POST   /v1/users             → 201
DELETE /v1/users/{user_id}   → 204

# 統一エラーレスポンス
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "User with id xxx not found",
    "request_id": "req_abc123"
  }
}
```

---

## コード/設計例（FastAPI）

```python
from fastapi import FastAPI, Depends, HTTPException, Request
from fastapi.responses import JSONResponse

app = FastAPI()

# 統一エラーハンドラ
@app.exception_handler(HTTPException)
async def http_exception_handler(request: Request, exc: HTTPException):
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": {"code": exc.detail, "request_id": request.state.request_id}}
    )

# バージョン付きルーター
from fastapi import APIRouter
v1 = APIRouter(prefix="/v1")

@v1.get("/users/{user_id}")
async def get_user(user_id: str, current_user=Depends(verify_firebase_token)):
    # user_id はUUID。連番整数は公開しない
    ...

app.include_router(v1)
```

---

## トレードオフ

| 選択 | メリット | デメリット |
|------|---------|-----------|
| URLバージョニング | 明確・デバッグしやすい | URLが冗長 |
| 厳しいレート制限 | 安定性が上がる | 正規ユーザーへの影響リスク |
| GraphQL採用 | フロント自由度が高い | サーバー負荷・N+1・キャッシュが複雑 |
| エラー詳細を返す | デバッグ容易 | 攻撃者に情報を渡す可能性 |

---

## チェックリスト

- [ ] エラーレスポンスのスキーマが全エンドポイントで統一されているか
- [ ] IDに連番整数を使っていないか（UUIDまたはULID推奨）
- [ ] レート制限が設定され、429 + Retry-After を返すか
- [ ] バージョニング戦略が決まっており、破壊的変更の手順があるか
- [ ] サービス間認証とユーザー認証を混同していないか（Firebase JWT vs サービスアカウント）
