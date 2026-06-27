# API設計のベストプラクティス：REST vs GraphQL・バージョニング・レート制限・認証

## 概要

APIはシステムの「契約」であり、一度公開したら簡単に変更できない。  
設計ミスはすべての利用者に影響し、後から直すコストが指数関数的に増大する。  
「とりあえず動くAPI」ではなく「変更・スケール・障害に強いAPI」を最初から設計する力が、AI時代のエンジニアに求められる。

---

## 仕組みの要点

### REST vs GraphQL 選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いている場面 | リソース単位操作、キャッシュ重視 | 複雑なネスト取得、UI主導 |
| N+1問題 | 複数エンドポイント必要 | DataLoaderで解決可能 |
| キャッシュ | HTTPキャッシュそのまま使える | クエリごとにキー設計が必要 |
| スキーマ管理 | OpenAPIで別途定義 | スキーマファーストで自動化 |

**FastAPIスタックでの原則**: CRUD + 単純なリソースはREST、複雑なフィルタ/集計はGraphQL。混在させない。

### バージョニング戦略

- **URLパス方式**（推奨）: `/api/v1/users` → シンプルで可視性高い
- **ヘッダー方式**: `Accept: application/vnd.api.v2+json` → URLが綺麗だがデバッグ困難
- **クエリパラメータ**: `?version=2` → 非推奨（キャッシュ汚染リスク）

**原則**: v1を壊す変更をするときだけv2を切る。フィールド追加は後方互換なので不要。

### レート制限設計

- **アルゴリズム選択**: Fixed Window（シンプル）< Sliding Window < Token Bucket（バースト許容）
- **対象単位**: IPアドレス < ユーザーID < APIキー（精度が上がるほどバイパスされにくい）
- **レスポンス**: 429 Too Many Requestsと`Retry-After`ヘッダーを必ず返す

### API認証パターン

| 方式 | 用途 | 注意点 |
|------|------|--------|
| Firebase JWT | ユーザー認証（BtoCアプリ） | 有効期限1h、Cloud Runでverify必須 |
| APIキー | サービス間・外部連携 | ローテーション機能を必ず用意 |
| mTLS | 高セキュリティなサービス間 | 証明書管理コストが高い |

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| `/getUser?id=1`（動詞をURLに含める） | `GET /users/1` |
| エラー全部200で返す | 適切なHTTPステータスコード（400/401/403/404/422/500） |
| `v1`のまま破壊的変更 | バージョンを上げるかフィールド追加で対応 |
| エラーレスポンスにスタックトレース | `{"error": "invalid_input", "message": "..."}` のみ返す |
| レート制限なしで公開 | Cloud RunのConcurrencyとCloudflare RateLimitを両方設定 |

---

## コード例（FastAPI + レート制限 + Firebase Auth）

```python
from fastapi import FastAPI, Depends, HTTPException, Request
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
app = FastAPI()

async def verify_firebase_token(request: Request) -> dict:
    token = request.headers.get("Authorization", "").removeprefix("Bearer ")
    if not token:
        raise HTTPException(401, "Missing token")
    try:
        return auth.verify_id_token(token)  # firebase_admin
    except Exception:
        raise HTTPException(401, "Invalid token")

@app.get("/api/v1/users/{user_id}")
@limiter.limit("30/minute")
async def get_user(
    user_id: str,
    request: Request,
    claims: dict = Depends(verify_firebase_token)
):
    if claims["uid"] != user_id:
        raise HTTPException(403, "Forbidden")
    return {"id": user_id}
```

---

## トレードオフ

| 選択 | メリット | デメリット |
|------|---------|-----------|
| RESTのみ | シンプル・キャッシュしやすい | 画面ごとに複数リクエスト必要 |
| GraphQLのみ | 柔軟・型安全 | N+1・キャッシュ設計が複雑 |
| URLバージョニング | 可視性・デバッグ容易 | URLが長くなる |
| IPベースレート制限 | 実装簡単 | NAT環境で誤検知・バイパス容易 |

---

## チェックリスト

- [ ] 破壊的変更時はAPIバージョンを上げているか
- [ ] 全エンドポイントに認証チェックがあるか（Cloud Runで全外部公開になっていないか）
- [ ] エラーレスポンスが統一されているか（`error`/`message`フィールド）
- [ ] レート制限が設定され、429と`Retry-After`を返しているか
- [ ] OpenAPI（Swagger）スキーマが自動生成・最新状態に保たれているか
