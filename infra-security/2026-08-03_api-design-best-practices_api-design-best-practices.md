# API設計のベストプラクティス — REST vs GraphQL・バージョニング・レート制限・認証

## 概要

APIは「システムの契約」であり、一度公開すると変更コストが跳ね上がる。AI時代でもAPIの設計力は差別化要因であり続ける。
良いAPI設計とは「変更しやすく、壊れにくく、誤用されにくい」設計のことだ。
FastAPI + Firebase Auth + Cloud Run 構成では、認証・レート制限・バージョニングを最初から設計に組み込む必要がある。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| データ取得 | エンドポイント固定 → Over/Under fetch | クライアント指定 → ちょうど欲しい量 |
| キャッシュ | HTTP GET がそのままキャッシュ可能 | POST 中心のためキャッシュ難 |
| スキーマ型安全 | OpenAPI で後付け | ビルトイン（SDL） |
| モバイル帯域 | フィールド削減が難しい | 帯域節約しやすい |
| **向いているケース** | 公開 API・シンプルな CRUD | BFF 層・複雑な関係データ |

**結論**: FastAPI + Cloud Run のスタックでは、まず REST で設計し、「複数エンドポイントを1回で叩きたい」という明確な需要が出たら GraphQL を検討する。

### バージョニング戦略

- **URL パス方式** (`/v1/users`): 最も明示的、キャッシュしやすい → **推奨**
- ヘッダー方式 (`Accept: application/vnd.api+json;version=2`): 隠れやすく、デバッグしにくい → 非推奨
- クエリパラメータ方式 (`?version=1`): ブックマーク可能だがAPIの「主語」が濁る → 避ける

**破壊的変更の定義**:
- フィールドの削除 / 型の変更 / 必須パラメータの追加 → 新バージョン必須
- フィールドの追加 / オプションパラメータの追加 → 同バージョンで対応可

### レート制限の設計

- **トークンバケット方式**: 瞬間的なバーストを許容しつつ平均レートを制御 → API向き
- **固定ウィンドウ方式**: 実装が簡単だが境界に集中攻撃されるリスクがある
- 識別子の優先順位: `user_id` > `API_KEY` > `IP` の順で精度が高い

レート制限はレスポンスヘッダーで明示する:
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1722643200
```

### 認証の設計

- Firebase Auth が発行する ID トークン（JWT）を `Authorization: Bearer <token>` で渡す
- バックエンドは毎回 Firebase Admin SDK で検証する（ローカルキャッシュ可）
- サービス間通信（Cloud Run 内部）は OIDC トークンを使い、Firebase Auth を経由しない

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|----------------|------------|
| `/getUser`・`/deleteUser` のような動詞URL | `/users/{id}` + HTTP メソッド（GET/DELETE）で意味を持たせる |
| 全エラーを `200 OK` で返す | 4xx/5xx を適切に使い、ボディに `code` + `message` を持たせる |
| バージョニングなしで破壊的変更 | URL に `/v1/` を最初から入れる |
| レート制限なしで公開 | Cloud Run 手前に Cloud Endpoints か Cloudflare でレート制限 |
| 内部エラーをそのままクライアントに返す | ユーザー向けメッセージとログ用の詳細を分ける |

---

## コード/設計例（FastAPI）

```python
from fastapi import FastAPI, Depends, HTTPException, Request
from firebase_admin import auth
import time

app = FastAPI()

# レート制限の簡易実装（本番はRedisで管理）
_rate_store: dict[str, list[float]] = {}
RATE_LIMIT = 60  # 1分間に60リクエスト

def rate_limit(request: Request):
    ip = request.client.host
    now = time.time()
    window = [t for t in _rate_store.get(ip, []) if now - t < 60]
    if len(window) >= RATE_LIMIT:
        raise HTTPException(status_code=429, detail="Rate limit exceeded")
    _rate_store[ip] = window + [now]

async def verify_token(request: Request) -> dict:
    token = request.headers.get("Authorization", "").removeprefix("Bearer ")
    try:
        return auth.verify_id_token(token)
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")

# バージョニング付きエンドポイント
@app.get("/v1/users/{user_id}", dependencies=[Depends(rate_limit)])
async def get_user(user_id: str, claims: dict = Depends(verify_token)):
    if claims["uid"] != user_id:
        raise HTTPException(status_code=403, detail="Forbidden")
    return {"id": user_id, "email": claims.get("email")}
```

---

## トレードオフ

| 選択 | メリット | デメリット |
|------|---------|-----------|
| URL バージョニング | 明示的・キャッシュしやすい | URL が長くなる |
| GraphQL 採用 | 柔軟なデータ取得 | キャッシュ難・学習コスト |
| IP ベースのレート制限 | 実装が簡単 | NAT 環境で誤検知 |
| Firebase Auth 一本化 | 管理コストゼロ | ベンダーロックイン |

---

## チェックリスト

- [ ] URL に `/v1/` が含まれており、破壊的変更時は `/v2/` に移行できる
- [ ] 全エンドポイントでトークン検証 + 認可チェック（本人確認）を行っている
- [ ] レート制限ヘッダー（`X-RateLimit-*`）をレスポンスに含めている
- [ ] 4xx エラーにはユーザー向けの `message` と開発者向けの `code` を両方返している
- [ ] 内部エラー（DB 接続失敗など）の詳細をクライアントに漏らしていない
