# API設計のベストプラクティス（REST vs GraphQL・バージョニング・レート制限・認証）

## 概要

AI時代においてAPIは「サービスの境界線」であり、設計の善し悪しがシステム全体の保守性・スケーラビリティに直結する。
「とりあえず動くエンドポイント」ではなく、**変更に強く、壊れにくく、使いやすいAPIを最初から設計する力**が求められる。
FastAPI + Cloud Run のスタックでは、起動コストとステートレス設計が設計判断に影響する。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いているケース | リソースが明確、キャッシュ重要 | フロントが多様、N+1問題が多い |
| 複雑さ | シンプル・学習コスト低 | スキーマ管理・認可が複雑 |
| パフォーマンス | CDNキャッシュが効く | 単一エンドポイント・キャッシュしにくい |
| 認証・認可 | エンドポイント単位で制御しやすい | フィールド単位の認可が必要で複雑 |

**判断基準**：クライアントが1種類（Web/モバイル共通）かつリソース構造が安定 → REST。
クライアントが多様でフィールド選択が頻繁 → GraphQL。モノリスの初期段階はRESTで十分。

### バージョニング戦略

- **URLバージョニング** (`/v1/`, `/v2/`)：最も一般的。キャッシュが効き、ログ解析が容易
- **ヘッダーバージョニング** (`Accept: application/vnd.api.v2+json`)：URLが綺麗だがテストしにくい
- **廃止方針**：`Deprecation` ヘッダーで事前告知 → 6ヶ月の猶予期間を設けてから削除

### レート制限の設計

- **どこで実装するか**：Cloud Run の前段（Cloud Armor / Cloudflare）が最優先。アプリ内は補完
- **単位**：IP単位 < ユーザーID単位 < APIキー単位（認証済みなら後者を優先）
- **レスポンス**：`429 Too Many Requests` + `Retry-After` ヘッダー + `X-RateLimit-*` ヘッダー
- **Cloud Run 固有**：コールドスタートでレート制限の状態（Redis等）が初期化されるためインメモリ管理は不可

### 認証の設計

- Firebase Auth の IDトークン（JWT）を `Authorization: Bearer <token>` で受け取る
- Cloud Run のサービス間通信は **サービスアカウント + OIDC トークン**（Firebase Auth とは別物）
- エンドポイントの認証要否は**デフォルト認証必須**にし、公開エンドポイントを明示的に列挙する

---

## アンチパターン vs 正しい設計

### アンチパターン

```
❌ /getUser?id=123          # 動詞をURLに入れる
❌ /api/v1/users/delete/123 # メソッドをURLに入れる
❌ 200 OK でエラーを返す   # { "status": "error", "code": 200 }
❌ 全エンドポイントを公開にしてトークンをbodyで受け取る
❌ バージョン管理なしで破壊的変更を本番に投入
```

### 正しい設計

```
✅ GET    /v1/users/{id}     # リソース + HTTPメソッドで表現
✅ DELETE /v1/users/{id}     # 204 No Content を返す
✅ エラーは適切なHTTPステータス + RFC 7807形式のbody
✅ Authorization ヘッダーでトークン受信、デフォルト認証必須
✅ /v2 への移行時は /v1 を Deprecation ヘッダーで告知してから廃止
```

---

## コード/設計例（FastAPI）

```python
from fastapi import FastAPI, Depends, HTTPException, Header
from fastapi.responses import JSONResponse

app = FastAPI()

# デフォルト認証必須。公開エンドポイントのみ dependencies を外す
async def verify_token(authorization: str = Header(...)):
    token = authorization.removeprefix("Bearer ")
    # Firebase Admin SDK でトークン検証（省略）
    return {"uid": "user_123"}

@app.get("/v1/users/{user_id}")
async def get_user(user_id: str, user=Depends(verify_token)):
    return {"id": user_id, "name": "Alice"}

# レート制限超過時のレスポンス
@app.exception_handler(429)
async def rate_limit_handler(request, exc):
    return JSONResponse(
        status_code=429,
        headers={"Retry-After": "60", "X-RateLimit-Limit": "100"},
        content={"type": "rate_limit_exceeded", "detail": "Too many requests"},
    )
```

---

## トレードオフ

| 選択 | メリット | デメリット |
|------|----------|------------|
| URLバージョニング | 明示的・キャッシュ効く・テスト容易 | URLが長くなる |
| アプリ内レート制限 | 細かいロジック実装可 | Cloud Run ステートレスで状態共有にRedis必要 |
| GraphQL採用 | フレキシブル・N+1解消可 | 認可・キャッシュ・スキーマ管理のコスト大 |
| 破壊的変更を許容 | 開発速度UP | クライアント側が壊れる・信頼性低下 |

---

## チェックリスト

- [ ] HTTPメソッドを正しく使っている（GET=取得、POST=作成、PUT/PATCH=更新、DELETE=削除）
- [ ] エラーレスポンスに適切なHTTPステータスコードを返している（200でエラーを返していない）
- [ ] 認証はデフォルト必須にし、公開エンドポイントを明示的に管理している
- [ ] レート制限をアプリの前段（Cloud Armor / Cloudflare）で実装している
- [ ] 破壊的変更の前に `Deprecation` ヘッダーで告知期間を設けている
