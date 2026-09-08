# API設計のベストプラクティス（REST vs GraphQL、バージョニング、レート制限、認証）

## 概要

APIはシステムの「契約」であり、一度公開すると変更コストが高い。  
設計の誤りは後から直すたびに破壊的変更を引き起こし、クライアントとの互換性を壊す。  
AI時代でも「何をAPIとして公開し、どう守り、どうスケールさせるか」を判断できる力は不変の価値を持つ。  
FastAPI + Cloud Run の構成では、設計の良し悪しが直接コストと可用性に直結する。

---

## 仕組みの要点

### REST vs GraphQL の判断基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いているケース | リソース単位の操作、キャッシュ重視 | フロントが多様、モバイル帯域節約 |
| Over-fetch問題 | ある（全フィールド返す） | ない（必要なフィールドだけ取得） |
| キャッシュ | HTTP標準でそのまま使える | クライアント側実装が必要 |
| N+1問題 | 設計で回避可能 | DataLoader必須、見逃しやすい |
| 学習コスト | 低い | スキーマ設計力が必要 |
| FastAPIとの相性 | ◎（ネイティブ） | △（Strawberry等の追加ライブラリ） |

**判断基準**: クライアントが1種（BFF or モバイルアプリ等）→ REST。  
クライアントが多様でフィールド要求がバラバラ → GraphQL。

### バージョニング戦略

- **URLバージョニング** (`/v1/users`)：最もシンプル、推奨
- **ヘッダーバージョニング** (`Accept: application/vnd.api+json;version=2`)：URLを汚さないが発見しにくい
- **クエリパラメータ** (`?version=2`)：キャッシュが効きにくい

**基本原則**:
- v1は廃止まで動かし続ける（最低6ヶ月の移行期間）
- 後方互換性を壊す変更（フィールド削除・型変更）はバージョンアップ必須
- フィールド追加はバージョンアップ不要（クライアントは無視できる）

### レート制限の設計

- **単位**: IP単位、ユーザー単位、APIキー単位を組み合わせる
- **アルゴリズム**: Token Bucket（バースト許容）か Fixed Window（実装簡単）
- **返すべきヘッダー**:
  - `X-RateLimit-Limit`: 上限
  - `X-RateLimit-Remaining`: 残り
  - `Retry-After`: 制限解除までの秒数
- **ステータスコード**: 429 Too Many Requests

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---------------|-----------|
| `GET /deleteUser?id=1` | `DELETE /users/1` |
| エラーを全部200で返す | 4xx/5xx を正しく使う |
| パスワードをクエリパラメータに | Authorizationヘッダー or Body |
| バージョニングなしで破壊的変更 | `/v2/` に新版を作る |
| エラーメッセージにスタックトレース | 内部情報を隠し、エラーコードのみ返す |
| `?page=1&limit=1000` を無制限に許可 | max_limitをサーバーで強制する |

---

## コード/設計例（FastAPI）

```python
from fastapi import FastAPI, Depends, HTTPException, Request
from fastapi.responses import JSONResponse
import time

app = FastAPI()

# レート制限（簡易実装）
request_counts: dict[str, list[float]] = {}

def rate_limit(request: Request, limit: int = 60, window: int = 60):
    client_ip = request.client.host
    now = time.time()
    hits = [t for t in request_counts.get(client_ip, []) if now - t < window]
    if len(hits) >= limit:
        raise HTTPException(
            status_code=429,
            detail={"error": "rate_limit_exceeded", "retry_after": window},
            headers={"Retry-After": str(window), "X-RateLimit-Limit": str(limit)}
        )
    request_counts[client_ip] = hits + [now]

# バージョニング付きエンドポイント
@app.get("/v1/users/{user_id}", dependencies=[Depends(rate_limit)])
async def get_user_v1(user_id: str):
    return {"id": user_id, "version": "v1"}

# 標準エラーレスポンス形式
@app.exception_handler(HTTPException)
async def http_exception_handler(request, exc):
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": exc.detail, "status": exc.status_code}
    )
```

---

## トレードオフ

| 選択 | メリット | デメリット |
|------|---------|-----------|
| URLバージョニング | 直感的、キャッシュしやすい | URL増殖、メンテナンスコスト |
| GraphQL | Over-fetch解消 | N+1、キャッシュ複雑化 |
| 厳格なレート制限 | 安全、コスト予測可能 | 正規ユーザーへの影響リスク |
| 全員同じエンドポイント | シンプル | 用途別最適化が難しい |

---

## チェックリスト

- [ ] HTTPメソッドを語義通りに使っている（GET=取得、POST=作成、PATCH=部分更新）
- [ ] エラーレスポンスは一貫したJSON形式で、スタックトレースを含まない
- [ ] レート制限ヘッダー（`X-RateLimit-*`, `Retry-After`）を返している
- [ ] 破壊的変更時は `/v2/` に新版を作り、旧版は移行期間を設けて廃止
- [ ] 認証トークン（JWT/Firebase IDトークン）はAuthorizationヘッダーで受け取る
