# API設計のベストプラクティス（REST vs GraphQL・バージョニング・認証）

## 概要

AI時代においてAPIは「システム同士をつなぐ契約」であり、一度公開すると変更コストが非常に高い。  
設計品質はシステムの保守性・スケーラビリティ・セキュリティに直結する。  
「とりあえず動くAPI」ではなく「変更に強く、壊れにくい設計」を最初から考えることが重要。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| ユースケース | リソース指向・CRUD中心 | 柔軟なクエリ・複数リソース結合 |
| クライアント多様性 | 少ない（1-2種類） | 多い（Web/Mobile/BFF） |
| N+1問題 | URLで1エンドポイント1クエリ | DataLoaderで解決必須 |
| キャッシュ | HTTP標準で容易 | クエリ毎に異なり難しい |
| 学習コスト | 低い | 高い（スキーマ・リゾルバ設計） |

**判断基準**: 社内API・シンプルCRUD → REST。モバイル+Webで同一バックエンド → GraphQL検討

### RESTful設計の原則

- **リソース名は名詞・複数形**: `/users`, `/orders/{id}` （動詞NG: `/getUser`）
- **HTTPメソッドで操作を表す**: GET(取得) / POST(作成) / PUT(全更新) / PATCH(部分更新) / DELETE
- **ステータスコードを正確に使う**: 200/201/400/401/403/404/409/422/500
- **レスポンスは一貫した構造**: `{data, error, meta}` を統一

### バージョニング戦略

- **URLパス方式** (`/v1/users`): シンプルで最も普及。推奨
- **ヘッダー方式** (`Accept: application/vnd.api+v1`): URLを汚さないが複雑
- **クエリパラメータ** (`?version=1`): 非推奨（キャッシュが壊れやすい）

**重要原則**:
- v1を廃止する場合は最低6ヶ月の猶予期間
- Deprecation-Date ヘッダーで廃止予定を通知
- バージョンを増やすより後方互換を保つ設計を優先

---

## アンチパターン vs 正しい設計

### アンチパターン

```
# NG: 動詞URL・エラー詳細なし・一貫性のないレスポンス
POST /createUser          # 動詞URL
GET  /getUserById?id=1    # クエリパラメータで識別子
→ 200 {"success": false}  # エラーなのに200
→ 500 {"error": "DB error: password column"} # 内部情報露出
```

### 正しい設計

```
# OK: リソース指向・適切なステータスコード・統一レスポンス
POST   /v1/users          → 201 {"data": {"id": "...", "email": "..."}}
GET    /v1/users/{id}     → 200 {"data": {...}}
PATCH  /v1/users/{id}     → 200 {"data": {...}}
DELETE /v1/users/{id}     → 204 (No Content)

# エラーレスポンス統一形式
→ 422 {
    "error": {
      "code": "VALIDATION_ERROR",
      "message": "入力値が不正です",
      "details": [{"field": "email", "reason": "形式が不正"}]
    }
  }
```

---

## コード/設計例（FastAPI + Cloud Run）

```python
from fastapi import APIRouter, HTTPException, Depends
from pydantic import BaseModel

router = APIRouter(prefix="/v1/users", tags=["users"])

class UserCreate(BaseModel):
    email: str
    name: str

@router.post("", status_code=201)
async def create_user(body: UserCreate, uid: str = Depends(verify_firebase_token)):
    # Firebase AuthでUID検証済み・入力はPydanticでバリデーション済み
    existing = await db.users.find_by_email(body.email)
    if existing:
        raise HTTPException(status_code=409, detail={"code": "EMAIL_CONFLICT"})
    user = await db.users.create(email=body.email, name=body.name, created_by=uid)
    return {"data": user}

@router.get("/{user_id}")
async def get_user(user_id: str, uid: str = Depends(verify_firebase_token)):
    user = await db.users.find(user_id)
    if not user:
        raise HTTPException(status_code=404, detail={"code": "NOT_FOUND"})
    return {"data": user}
```

---

## レート制限・認証の設計

### レート制限

- **エンドポイント別に設定**: 書き込みは厳しく・読み取りは緩く
  - `POST /users`: 10 req/min/IP
  - `GET /users`: 100 req/min/token
- **レスポンスヘッダーで状態通知**: `X-RateLimit-Remaining`, `Retry-After`
- **Cloud Run + Cloudflare WAF** で多層防御

### 認証フロー（Firebase Auth + FastAPI）

```
Client → Firebase Auth → IDトークン取得
Client → FastAPI (Authorization: Bearer <token>)
FastAPI → firebase_admin.auth.verify_id_token() → UID取得
FastAPI → DBクエリ (WHERE user_id = UID)
```

---

## トレードオフ

| 設計選択 | メリット | デメリット |
|---------|---------|---------|
| URLバージョニング | シンプル・キャッシュ効く | URL肥大化 |
| GraphQL | 柔軟なクエリ | N+1・キャッシュ難 |
| ページネーション(カーソル型) | 大量データに強い | 実装複雑 |
| ページネーション(オフセット型) | シンプル | 大量データで遅い |
| エラー詳細を返す | デバッグ容易 | 内部情報露出リスク |

---

## チェックリスト

- [ ] URLはリソース名詞・複数形で統一されているか
- [ ] HTTPステータスコードが意味通りに使われているか（エラーに200を返していないか）
- [ ] エラーレスポンスの形式が全エンドポイントで統一されているか
- [ ] バージョニング戦略が決まっており、廃止ポリシーが明文化されているか
- [ ] レート制限が実装され、`X-RateLimit-*` ヘッダーで通知しているか
