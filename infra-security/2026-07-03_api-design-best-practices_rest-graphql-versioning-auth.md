# API設計のベストプラクティス — REST vs GraphQL・バージョニング・レート制限・認証

## 概要

APIはシステムの「契約」であり、一度公開すると変更コストが極めて高い。
AI時代においても「どういうAPIを設計するか」はコードを書く力よりも重要なスキルになっている。
設計ミスは障害・セキュリティ脆弱性・技術的負債の三重苦を招く。
FastAPI + Cloud Run + Firebase Auth スタックで正しい設計判断ができることを目指す。

---

## 仕組みの要点

### REST vs GraphQL — 選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いているケース | リソース操作が明確、外部公開API | フロントが多様、過剰取得/不足取得を避けたい |
| キャッシュ | HTTPキャッシュがそのまま使える | クエリごとにハッシュが必要（複雑） |
| 型安全 | OpenAPI/Swagger | スキーマが型定義を兼ねる |
| 学習コスト | 低い | 高い（N+1問題, DataLoaderが必要） |
| **FastAPI向き** | ✅ ほぼREST一択 | 別ライブラリ(Strawberry)が必要 |

**判断則**: チームが小さい・外部向けAPIがある・キャッシュが重要 → REST。社内向けで画面が多様 → GraphQLを検討。

### エンドポイント設計の原則

- **リソース名は名詞・複数形**: `/users/{id}` ✅ 、`/getUser` ❌
- **HTTPメソッドで動詞を表現**: GET(取得) / POST(作成) / PUT(全更新) / PATCH(部分更新) / DELETE(削除)
- **ネストは2階層まで**: `/users/{id}/orders` ✅ 、`/users/{id}/orders/{id}/items/{id}` ❌
- **フィルタ・ソートはクエリパラメータ**: `/orders?status=pending&sort=-created_at`

### バージョニング戦略

3つのアプローチと推奨:

- **URLパス方式** `/api/v1/users` → 最もシンプル・視認性高い **← 推奨**
- **ヘッダー方式** `API-Version: 2024-01-01` → クリーンだがキャッシュ困難
- **クエリパラメータ** `/users?version=1` → 避ける（ログが汚れる）

**廃止ポリシーを最初に決める**: `v1`のサポート終了日をドキュメントに明記し、`Deprecation`ヘッダーで警告を返す。

---

## アンチパターン vs 正しい設計

### ❌ アンチパターン

```
GET  /getUsers          # 動詞をURLに含める
POST /updateUser/123    # POSTで更新
GET  /users?page=0&perPage=999999  # 上限なしページネーション
# エラーレスポンスが全部 { "error": "something went wrong" }
# 全エンドポイントが認証なし → 後付けでトークン追加
```

### ✅ 正しい設計（FastAPI例）

```python
from fastapi import APIRouter, Depends, HTTPException, Query
from typing import Annotated

router = APIRouter(prefix="/api/v1/users", tags=["users"])

@router.get("", response_model=PaginatedResponse[UserSchema])
async def list_users(
    page: Annotated[int, Query(ge=1)] = 1,
    limit: Annotated[int, Query(ge=1, le=100)] = 20,  # 上限を強制
    current_user: User = Depends(get_current_user),    # 認証必須
):
    ...

@router.patch("/{user_id}", response_model=UserSchema)
async def update_user(user_id: str, body: UserUpdateSchema, ...):
    # 404 Not Found / 403 Forbidden を明確に分ける
    raise HTTPException(status_code=404, detail={"code": "USER_NOT_FOUND"})
```

---

## レート制限の設計

- **単位**: ユーザーID > APIキー > IP（この優先順位で制限）
- **アルゴリズム**: Token Bucket（バースト許容）が現実的。Sliding Windowは精度高いが実装重い
- **レスポンスヘッダー**で残量を返す: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`
- **ステータスコード**: `429 Too Many Requests` + `Retry-After` ヘッダー
- **Cloud Runでの実装**: Redisを使ったトークンバケット or Cloudflare Rate Limiting Rules（アプリ外で処理）

## 認証設計

- **Firebase Auth + JWT**: `Authorization: Bearer <token>` → Cloud RunでFirebase Admin SDKで検証
- **サービス間通信**: Google OIDC トークン（`Authorization: Bearer <service_account_token>`）
- **APIキー**: ユーザー向け長期トークンが必要な場合のみ。DBに`hashed_api_key`で保存

---

## トレードオフ

| 選択 | メリット | デメリット |
|------|---------|-----------|
| REST | シンプル・キャッシュ容易 | 複数リソースを取るとN回リクエスト |
| URLバージョニング | 明示的・ルーティング容易 | URL設計が変わるとリンク切れ |
| 厳格なレート制限 | 安全・コスト予測可能 | 正当なユーザーをブロックするリスク |
| 認証を全エンドポイントに | セキュア | ヘルスチェックにも認証が必要になりがち |

---

## チェックリスト

- [ ] エンドポイントは名詞複数形・HTTPメソッドで動詞を表現している
- [ ] ページネーションに上限（`limit <= 100`など）を設けている
- [ ] エラーレスポンスに `code` フィールド（機械可読）を含めている
- [ ] バージョン廃止ポリシーをドキュメントに明記している
- [ ] レート制限を「認証ユーザー単位」で実装し、429+Retry-Afterを返している
