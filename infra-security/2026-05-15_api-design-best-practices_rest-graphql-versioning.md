# API設計のベストプラクティス：REST vs GraphQL、バージョニング、レート制限、認証

## 概要

APIはシステムの「契約」であり、一度公開したら簡単には変更できない。
AI時代でも「設計が悪いAPIは、いくら実装が綺麗でも運用コストが高い」という事実は変わらない。
スケール・後方互換性・セキュリティを最初に織り込んだ設計が、長期的な運用コストを左右する。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いている用途 | リソース単位のCRUD、外部公開API | 複雑なリレーション、BFF、モバイル |
| キャッシュ | URL単位でCDNキャッシュ可能 | POST固定でキャッシュ難 |
| 型安全 | OpenAPI/Swaggerで補う | スキーマ定義が強制される |
| 学習コスト | 低い | クライアント・サーバー両方高い |
| N+1問題 | ない（エンドポイント設計で回避） | DataLoaderが必須 |

**判断の原則：**
- 外部向けAPI、シンプルなCRUD → REST
- 内部BFF、フロントの型安全重視、フィールド選択が多様 → GraphQL
- どちらでもない理由でGraphQLを選ばない

---

## アンチパターン vs 正しい設計

### アンチパターン

```
# バージョニングなしで破壊的変更
GET /api/users → フィールド削除・型変更を平気でやる

# 動詞をURLに入れる
POST /api/getUser
POST /api/deleteUser

# エラーレスポンスが不統一
{"error": "not found"}  # あるエンドポイント
{"message": "404"}      # 別のエンドポイント
{"status": "fail"}      # また別のエンドポイント
```

### 正しい設計

```
# URLにバージョン or ヘッダーバージョニング
GET /v1/users/{id}
Accept: application/vnd.myapi.v2+json

# リソース + HTTPメソッドで表現
GET    /users/{id}     # 取得
POST   /users          # 作成
PATCH  /users/{id}     # 部分更新
DELETE /users/{id}     # 削除

# エラーレスポンスを統一（RFC 7807 Problem Details）
{"type": "https://api.example.com/errors/not-found",
 "title": "User not found", "status": 404, "detail": "..."}
```

---

## FastAPI + Cloud Run での実装例

```python
from fastapi import FastAPI, Depends, HTTPException, Request
from fastapi.responses import JSONResponse
import time

app = FastAPI(title="API", version="1.0")

# レート制限（シンプル実装。本番はRedisで共有状態を持つ）
request_counts: dict = {}

async def rate_limit(request: Request):
    client_ip = request.client.host
    now = int(time.time() // 60)  # 1分window
    key = f"{client_ip}:{now}"
    request_counts[key] = request_counts.get(key, 0) + 1
    if request_counts[key] > 100:
        raise HTTPException(status_code=429, headers={"Retry-After": "60"})

# バージョニング付きルーター
from fastapi import APIRouter
v1 = APIRouter(prefix="/v1", dependencies=[Depends(rate_limit)])

@v1.get("/users/{user_id}")
async def get_user(user_id: str):
    return {"id": user_id, "version": "v1"}

app.include_router(v1)
```

---

## バージョニング戦略

| 方式 | 例 | メリット | デメリット |
|------|-----|---------|-----------|
| URLパス | `/v1/users` | シンプル・CDNキャッシュ容易 | URLが変わる |
| ヘッダー | `Accept-Version: v2` | URLが汚れない | 可視性が低い |
| クエリパラメータ | `?version=2` | 手軽 | キャッシュ汚染リスク |

**推奨：** 外部公開APIはURLパスバージョニング。変更時は**最低6ヶ月間旧バージョンを維持**し、Sunset ヘッダーで廃止日を通知。

---

## 認証の設計パターン（Firebase Auth + Cloud Run）

```
クライアント
  │─── Firebase ID Token (Bearer) ───▶ Cloud Run (FastAPI)
                                            │
                                    firebase_admin.auth.verify_id_token()
                                            │
                                    ユーザーIDを取得 → DB照合 → レスポンス
```

- サービス間認証は **Google-signed OIDC token**（Cloud Run identity）
- IDトークンは**毎リクエスト検証**（キャッシュは最大5分）
- スコープ（権限）はAPIレイヤーではなく**ミドルウェアで統一検証**

---

## トレードオフ

| 選択 | メリット | コスト |
|------|---------|--------|
| 厳格なバージョニング | クライアント互換性保証 | 旧バージョン維持コスト |
| GraphQL | 柔軟なクエリ | N+1対策・キャッシュ設計の複雑さ |
| 厳密なレート制限 | 悪用防止・コスト保護 | Redis等の追加インフラ |
| RFC 7807エラー形式 | クライアント実装が楽 | 初期設計コスト |

---

## チェックリスト

- [ ] エラーレスポンスのフォーマットが全エンドポイントで統一されているか
- [ ] URLにバージョンが含まれているか（または廃止戦略があるか）
- [ ] レート制限が実装されており、429レスポンスに `Retry-After` ヘッダーがあるか
- [ ] 認証トークンはリクエストごとに検証されているか（キャッシュ時間が適切か）
- [ ] 破壊的変更（フィールド削除・型変更）を加える前に旧バージョンの廃止予告をしているか
