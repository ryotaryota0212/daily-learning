# API設計のベストプラクティス（REST vs GraphQL、バージョニング、レート制限、認証）

## 概要

API設計はシステムの「契約」であり、一度公開すると変更コストが非常に高い。
設計ミスは後続チームやクライアントに波及し、技術的負債として長期間残る。
「動くAPI」ではなく「壊れにくく、変更に強く、セキュアなAPI」を設計する力が問われる。
FastAPI + Cloud Run + Firebase Auth スタックにおいて、設計ミスを防ぐ実践的な知識をまとめる。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|---|---|---|
| クライアントの多様性 | 少ない（1種類のUI） | 多い（Web/Mobile/外部） |
| データ取得の柔軟性 | 固定レスポンス | クライアント指定 |
| キャッシュのしやすさ | GET はCDNキャッシュ可 | 原則POST、キャッシュ困難 |
| 学習コスト | 低い | 高い（スキーマ設計必要） |
| 適した場面 | 内部API、シンプルCRUD | BFF、フロント主導のプロダクト |

**結論**: 迷ったらREST。GraphQLは「同じエンドポイントから異なる形式でデータを取る需要」が明確にある場合のみ選ぶ。

### RESTのリソース設計原則

- URL は名詞（リソース）で構成する：`/users/{id}/orders` ✓、`/getOrdersByUser` ✗
- HTTPメソッドでアクションを表す：GET（取得）、POST（作成）、PUT/PATCH（更新）、DELETE（削除）
- ステータスコードを正しく返す：200/201/204/400/401/403/404/409/422/500
- コレクション操作はページネーション必須：`?cursor=xxx&limit=20`（オフセットよりカーソルが安全）
- エラーレスポンスは一貫した形式に：`{ "error": { "code": "INVALID_INPUT", "message": "..." } }`

---

## アンチパターン vs 正しい設計

### アンチパターン

```
# 悪いURL設計
GET /api/getUserData?userId=123&type=orders
POST /api/createOrUpdateUser  # 動詞入りURL
GET /api/v1/orders?page=1&pageSize=1000  # 大量オフセットページネーション
```

- バージョンを URL に入れない → `/api/users` で後から変更できなくなる
- 認証なしで内部APIを公開する → Cloud Run の「未認証を許可」設定ミスで露出
- エラー時に 200 OK + body にエラー情報を返す → クライアントが検知できない

### 正しい設計

```python
# FastAPI での正しい設計例
from fastapi import APIRouter, Depends, HTTPException, Query
from typing import Optional

router = APIRouter(prefix="/v1/users", tags=["users"])

@router.get("/{user_id}/orders")
async def list_orders(
    user_id: str,
    cursor: Optional[str] = Query(None),
    limit: int = Query(default=20, le=100),  # 上限を強制
    current_user=Depends(verify_firebase_token),  # 認証必須
):
    if current_user.uid != user_id:  # 認可チェック
        raise HTTPException(status_code=403, detail="Forbidden")
    return await fetch_orders(user_id, cursor, limit)
```

---

## バージョニング戦略

- **URLバージョニング**（推奨）: `/v1/users`, `/v2/users` — 最も明示的でキャッシュしやすい
- **ヘッダーバージョニング**: `API-Version: 2` — URLを汚さないが、テストしづらい
- **破壊的変更のルール**: フィールド追加はOK、フィールド削除/型変更は新バージョン

**移行戦略**: 旧バージョンを最低6ヶ月維持し、Deprecation-Date ヘッダーで通知する。

---

## レート制限の設計

```python
# Cloud Run + Redis でのレート制限イメージ
# ヘッダーで状態を返すことでクライアントが自律的に制御できる
async def rate_limit_middleware(request: Request, call_next):
    key = f"rate:{get_client_id(request)}"
    count = await redis.incr(key)
    if count == 1:
        await redis.expire(key, 60)  # 60秒ウィンドウ
    if count > 100:  # 100req/min
        return JSONResponse(status_code=429,
            headers={"Retry-After": "60",
                     "X-RateLimit-Limit": "100",
                     "X-RateLimit-Remaining": "0"})
    response = await call_next(request)
    response.headers["X-RateLimit-Remaining"] = str(100 - count)
    return response
```

- 認証ユーザーと未認証ユーザーで制限を分ける（未認証は厳しく）
- エンドポイントごとに異なる制限を設ける（検索は厳しく、読み取りは緩く）
- 429 Too Many Requests + `Retry-After` ヘッダーで返す

---

## 認証・認可の設計

- **認証（Authentication）**: Firebase Auth の ID トークンを `Authorization: Bearer <token>` で受け取る
- **認可（Authorization）**: トークン検証後、リソースオーナーかどうかを確認する（必ずアプリ層で実施）
- Cloud Run のサービス間通信: IAM サービスアカウント + ID トークンで認証（API キーは使わない）

```python
# Firebase トークン検証
async def verify_firebase_token(authorization: str = Header(...)):
    token = authorization.replace("Bearer ", "")
    try:
        decoded = firebase_admin.auth.verify_id_token(token)
        return decoded  # uid, email などが入っている
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")
```

---

## トレードオフ

| 選択 | メリット | デメリット |
|---|---|---|
| REST | シンプル、CDNキャッシュ可 | オーバーフェッチ/アンダーフェッチ |
| GraphQL | 柔軟なデータ取得 | キャッシュ困難、N+1問題 |
| URLバージョニング | 明示的、キャッシュ容易 | URLが増える |
| カーソルページネーション | 大量データで安全 | 「n件目から」の直接指定が難しい |
| レート制限(IPベース) | 実装が簡単 | 共有IPで誤検知が起きやすい |

---

## チェックリスト

- [ ] URLは名詞ベースで設計し、HTTPメソッドでアクションを表現しているか
- [ ] すべてのエンドポイントに認証（Firebase Auth）と認可（オーナーチェック）を実装しているか
- [ ] レート制限を実装し、429 + Retry-After ヘッダーを返しているか
- [ ] APIバージョンをURLに含め、破壊的変更は新バージョンに切り出しているか
- [ ] エラーレスポンスの形式が統一されており、適切なHTTPステータスコードを返しているか
