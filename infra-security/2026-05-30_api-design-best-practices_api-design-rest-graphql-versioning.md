# API設計のベストプラクティス（REST vs GraphQL・バージョニング・認証）

## 概要
APIはシステムの「契約」であり、一度公開すると変更コストが非常に高い。
REST vs GraphQLの選択ミス、バージョニング戦略の欠如、認証設計の甘さは後から修正困難。
FastAPI + Firebase Auth環境で「壊れにくいAPI」を設計するための判断基準をまとめる。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いているケース | CRUD中心・シンプルな操作 | 複雑な関連データ取得 |
| クライアントの自由度 | 低い（サーバー側で定義） | 高い（必要なフィールドだけ取得） |
| HTTPキャッシュ | 使いやすい（GETが効く） | 難しい（基本的にPOSTのみ） |
| Over-fetch / Under-fetch | 起きやすい | 解決できる |
| チーム学習コスト | 低い | 高い |

**推奨判断**: スタートアップ・小規模チームはREST一択。複数クライアント（Web/Mobile）が異なる形でデータを必要とするときだけGraphQLを検討。

### バージョニング戦略

- **URLパス方式（推奨）**: `/api/v1/users` — シンプルで一目瞭然
- **ヘッダー方式**: `Accept-Version: v2` — URLがクリーンだが実装複雑
- **非推奨**: クエリパラメータ方式（`?version=1`）— キャッシュと相性が悪い
- 後方互換を壊す変更時だけメジャーバージョンを上げる

### エンドポイント設計の原則

- リソース名は**複数形の名詞**: `/users`, `/orders`, `/products`
- **HTTPメソッドでアクションを表現**:
  - `GET /users` → 一覧, `GET /users/{id}` → 単体取得
  - `POST /users` → 作成, `PUT/PATCH /users/{id}` → 更新
  - `DELETE /users/{id}` → 削除
- ネストは2階層まで: `/users/{id}/orders` はOK、それ以上は再設計を検討
- フィルタ・ソートはクエリパラメータ: `GET /users?status=active&sort=created_at`

### 認証の設計（Firebase Auth前提）

- Firebase AuthのJWTをBearerトークンとして受け取る
- **ユーザーIDはJWTの`sub`（uid）から取得** → リクエストBodyの`user_id`は絶対に信用しない
- 全エンドポイントにDependsで認証を強制し、公開エンドポイントだけ明示的に除外

### エラーレスポンスの統一

```json
{
  "error": "VALIDATION_ERROR",
  "message": "メールアドレスの形式が正しくありません",
  "detail": [{"field": "email", "issue": "invalid_format"}]
}
```

- HTTPステータスコードを正しく使う: `200/201/400/401/403/404/409/422/500`
- `error`フィールドは機械可読な定数文字列（クライアントがswitch文で処理できる）

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| `POST /getUser` | `GET /users/{id}` |
| エラーを全て200で返す | 適切なHTTPステータス + エラーコード |
| Bodyの`user_id`を信用する | JWTの`sub`からuidを取得 |
| エラー形式がエンドポイントごとにバラバラ | 統一されたエラーレスポンス構造 |
| バージョンなしで破壊的変更 | URLバージョニングで段階的廃止 |

---

## コード/設計例

```python
from fastapi import Depends, HTTPException
from fastapi.security import HTTPBearer
from firebase_admin import auth

bearer = HTTPBearer()

async def get_current_user(token=Depends(bearer)):
    try:
        decoded = auth.verify_id_token(token.credentials)
        return decoded  # uid, email 等が取得できる
    except Exception:
        raise HTTPException(status_code=401, detail={"error": "INVALID_TOKEN"})

@app.get("/api/v1/users/me")
async def get_me(user=Depends(get_current_user)):
    return {"uid": user["uid"], "email": user.get("email")}

# 統一エラーハンドラ
@app.exception_handler(RequestValidationError)
async def validation_handler(req, exc):
    return JSONResponse(status_code=422, content={
        "error": "VALIDATION_ERROR",
        "detail": exc.errors()
    })
```

---

## トレードオフ

| 選択 | メリット | デメリット |
|------|---------|-----------|
| REST（GraphQL不使用） | シンプル・キャッシュ効く | 複雑なデータ取得でN+1問題 |
| URLバージョニング | 一目で判別、ルーティング楽 | URL変更でクライアント更新が必要 |
| 全エンドポイント認証必須化 | セキュア・穴が生まれにくい | パブリックAPIの追加が少し手間 |
| 統一エラーレスポンス | クライアント実装が安定 | 初期設計コストが必要 |

---

## チェックリスト
- [ ] リソース名が複数形の名詞で、HTTPメソッドが適切に使われているか
- [ ] 認証はJWTの`uid`から取得し、リクエストのuser_idを信用していないか
- [ ] エラーレスポンスが統一された形式か（HTTPステータス + 定数エラーコード）
- [ ] APIバージョン戦略を決め、破壊的変更の手順をドキュメント化しているか
- [ ] ネストが2階層以内、クエリパラメータでフィルタ・ソートを実装しているか
