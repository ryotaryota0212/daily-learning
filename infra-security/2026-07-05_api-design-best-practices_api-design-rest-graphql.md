# API設計のベストプラクティス：REST vs GraphQL・バージョニング・レート制限・認証

## 概要

APIはシステムの「契約」であり、一度公開すると変更が難しい。設計ミスは長期の技術的負債になる。
FastAPI + Firebase Auth + Cloud Run スタックを前提に、REST vs GraphQLの選択基準、安全なバージョニング、
レート制限、認証の正しい設計を習得する。AI時代においても「APIの境界設計」はエンジニアが担う核心的な責務。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| リソースが明確 | ◎ | △ |
| クライアントが多様（Web/Mobile） | △ | ◎ |
| HTTPキャッシュを活用したい | ◎ | △（POSTベース） |
| スキーマ駆動開発 | △ | ◎ |
| チームがシンプルさを好む | ◎ | △ |

**判断原則**: 迷ったらRESTから始める。GraphQLへの移行は後からできる。N+1問題とキャッシュ設計が複雑になるため、GraphQLは明確な理由があるときだけ選ぶ。

### バージョニング戦略

| 方式 | 例 | 推奨度 | 理由 |
|------|------|--------|------|
| URLパス | `/api/v1/users` | ◎ | 明確・キャッシュしやすい |
| ヘッダー | `API-Version: 2` | △ | 見えにくい |
| クエリパラメータ | `?version=2` | △ | キャッシュが効きにくい |

破壊的変更は必ず新バージョン（`/v2/`）で行い、旧バージョンは最低6ヶ月は維持する。

### レート制限の設計

- **対象単位**: ユーザーID > APIキー > IPアドレスの優先順位
- **アルゴリズム**:
  - Token Bucket: バーストを許可しつつ長期的に制限（推奨）
  - Sliding Window: 均等に制御したい場合
- **レスポンス**: `429 Too Many Requests` + `Retry-After` ヘッダー必須
- **FastAPIでの実装**: CloudflareのWAF or Cloud Runのサイドカーで実装が現実的

### 認証の設計

- Firebase Auth JWT → `Authorization: Bearer <token>` → Cloud Run Middleware で検証
- サービス間通信はIDトークンではなくCloud Runのサービスアカウント認証を使う
- APIキーを使う場合はハッシュ化してDBに保存（平文禁止）

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| `GET /getUserById?id=1` | `GET /users/{id}` |
| エラー全て `200 OK` で返す | HTTPステータスコードを正しく使う |
| バージョンなしで破壊的変更 | `/v2/` を新設し旧バージョンを維持 |
| エラーメッセージにスタックトレース含む | 本番では汎用メッセージのみ返す |
| レート制限なし | 認証済み: 1000 req/h、匿名: 100 req/h |
| 全エンドポイントをデフォルト公開 | デフォルト非公開・明示的に公開を宣言 |

---

## コード例（FastAPI）

```python
from fastapi import FastAPI, Depends, HTTPException, Request
from fastapi.responses import JSONResponse
from fastapi.security import HTTPBearer
from pydantic import BaseModel

app = FastAPI()
security = HTTPBearer()

# バージョニング：ルーターで分離
from fastapi import APIRouter
v1 = APIRouter(prefix="/api/v1", tags=["v1"])
v2 = APIRouter(prefix="/api/v2", tags=["v2"])

@v1.get("/users/{user_id}")
async def get_user_v1(user_id: str, token=Depends(security)):
    return {"id": user_id, "name": "legacy_field"}

@v2.get("/users/{user_id}")
async def get_user_v2(user_id: str, token=Depends(security)):
    return {"id": user_id, "display_name": "new_field"}  # 破壊的変更をv2で吸収

# エラーレスポンス標準化
class ErrorResponse(BaseModel):
    error: str
    code: str

@app.exception_handler(HTTPException)
async def http_exception_handler(request: Request, exc: HTTPException):
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": exc.detail, "code": "API_ERROR"}
    )

app.include_router(v1)
app.include_router(v2)
```

---

## トレードオフ

| 選択 | メリット | デメリット |
|---|---|---|
| URLバージョニング | 明確・キャッシュしやすい | URL変更が必要 |
| GraphQL採用 | 柔軟なフィールド取得 | N+1問題・キャッシュ複雑・学習コスト |
| 厳格なレート制限 | DoS対策・コスト保護 | 正規ユーザーへの影響リスク |
| 後方互換維持 | クライアント影響なし | 旧バージョン維持コスト増 |
| APIキー認証 | 実装シンプル | キーの漏洩リスク・ローテーション運用が必要 |

---

## チェックリスト

- [ ] HTTPメソッドとステータスコードがREST規約に従っている（`GET`は冪等、`DELETE`は`204`等）
- [ ] バージョニング戦略が決まっており、破壊的変更は必ず新バージョンで行う
- [ ] 全エンドポイントにレート制限が適用されている（`429` + `Retry-After` 返却）
- [ ] エラーレスポンスが標準化されており、スタックトレースや内部情報を漏らさない
- [ ] 認証が必要なエンドポイントはデフォルト保護・明示的に公開宣言する設計になっている
