# API設計のベストプラクティス（REST vs GraphQL、バージョニング、レート制限、認証）

## 概要

APIはシステムの「契約」であり、一度公開すると変更コストが高い。AI時代においても、外部システムやフロントエンドとの接続点はAPIが担う。「とりあえず動くエンドポイント」を増やすのではなく、「スケールしても壊れにくい設計」を最初から考えることが重要。FastAPI + Cloud Run + Firebase Auth スタックにおける実践的な設計指針をまとめる。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いているケース | CRUD中心・外部公開API | 複数リソースを柔軟に取得したいフロント |
| Over-fetching | 発生しやすい | 必要なフィールドだけ取得 |
| キャッシュ | CDN・HTTPキャッシュが効く | クエリが動的なためキャッシュ困難 |
| スキーマ管理 | OpenAPI/Swagger | GraphQL Schema（型安全） |
| チーム経験 | 広く普及 | 学習コストあり |

**判断原則**: 外部公開・シンプルなCRUDはREST。BFF（Backend for Frontend）や複雑なデータ結合が必要な内部APIはGraphQLを検討。

### バージョニング戦略

- **URLパスによるバージョン管理**（最も一般的）: `/api/v1/users`
- **ヘッダーバージョン**: `API-Version: 2`（URLを汚さないが発見性が低い）
- **クエリパラメータ**: `/users?version=2`（避けるべき）

**推奨**: v1→v2 移行時は最低6ヶ月間 v1 を維持。廃止予定は `Deprecation` ヘッダーで通知。

### レート制限の設計

- **識別単位**: IPアドレス < APIキー < ユーザーID（精度が高い順）
- **アルゴリズム選択**:
  - Fixed Window: 実装簡単、バースト問題あり
  - Sliding Window: 精度高い
  - Token Bucket: バーストを許容しつつ上限を保つ（推奨）
- **レスポンスヘッダーで残量を返す**: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`

---

## アンチパターン vs 正しい設計

### アンチパターン
- エンドポイントごとに認証ロジックをコピペ → ミドルウェアで一元化すべき
- `POST /getUser` のように動詞をURLに入れる → リソース指向（名詞）で設計
- エラーレスポンスが `{"error": "失敗しました"}` のみ → エラーコード・詳細・トレースIDを含める
- バージョン管理なし → v1から始め、破壊的変更は必ずバージョンアップ
- 全エンドポイントが認証不要 → デフォルト認証必須、例外を明示

### 正しい設計
- URLはリソース（名詞）中心: `GET /users/{id}`, `POST /orders`, `DELETE /sessions/{id}`
- HTTPステータスコードを正しく使う: 201 Created, 409 Conflict, 422 Unprocessable Entity
- エラーレスポンスを統一: `{"code": "USER_NOT_FOUND", "message": "...", "trace_id": "..."}`
- 認証・認可をミドルウェアに集約

---

## コード/設計例（FastAPI + Firebase Auth）

```python
from fastapi import Depends, HTTPException, Security
from fastapi.security import HTTPBearer
from firebase_admin import auth
import time

security = HTTPBearer()

# 認証ミドルウェア（全ルートで再利用）
async def get_current_user(token = Security(security)):
    try:
        decoded = auth.verify_id_token(token.credentials)
        return decoded
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")

# レート制限（簡易版：本番はRedis使用）
request_counts: dict = {}

async def rate_limit(user = Depends(get_current_user)):
    uid = user["uid"]
    now = int(time.time() // 60)  # 1分窓
    key = f"{uid}:{now}"
    request_counts[key] = request_counts.get(key, 0) + 1
    if request_counts[key] > 60:  # 60req/min
        raise HTTPException(status_code=429, detail="Rate limit exceeded")
    return user

# バージョン付きルーター
from fastapi import APIRouter
v1_router = APIRouter(prefix="/api/v1")

@v1_router.get("/users/{user_id}")
async def get_user(user_id: str, current_user = Depends(rate_limit)):
    # 認可チェック：自分のリソースのみ
    if current_user["uid"] != user_id:
        raise HTTPException(status_code=403, detail="Forbidden")
    return {"id": user_id}
```

---

## トレードオフ

| 選択 | メリット | デメリット |
|------|----------|-----------|
| URLバージョニング | 明示的で発見しやすい | URL設計が複雑になる |
| GraphQL | 柔軟なデータ取得 | N+1問題、キャッシュが難しい |
| Token Bucketレート制限 | バーストに寛容 | 実装にRedis等が必要 |
| Firebase Auth統合 | 実装コスト低 | ベンダーロックイン |

---

## チェックリスト

- [ ] エラーレスポンスに `code`・`message`・`trace_id` が含まれているか
- [ ] 認証ミドルウェアが全エンドポイントに適用されているか（除外は明示的に）
- [ ] レート制限のヘッダー（`X-RateLimit-*`）をレスポンスに含めているか
- [ ] APIバージョンが明示され、破壊的変更時はバージョンアップされているか
- [ ] OpenAPI（Swaggerドキュメント）が自動生成・最新の状態か
