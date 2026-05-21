# API設計のベストプラクティス — REST vs GraphQL、バージョニング、レート制限、認証

## 概要

APIはシステムの「契約」であり、一度公開すると変更コストが跳ね上がる。  
AI時代においても「どの形式で、どの粒度で、どう守るか」を設計できる力は不変のコアスキル。  
間違った設計は技術的負債ではなく「永続する傷」になる。FastAPI + Firebase Auth + Cloud Run 前提で整理する。

---

## 仕組みの要点

### REST vs GraphQL — 選択基準

| 観点 | REST | GraphQL |
|---|---|---|
| 向いているケース | リソースが明確、公開API | フロントが多様、過剰取得が問題 |
| キャッシュ | HTTP標準で容易 | 工夫が必要（POST中心のため） |
| 型安全 | OpenAPIで補完 | スキーマが強制 |
| N+1問題 | 設計次第で回避しやすい | DataLoaderなどで対処必須 |
| 学習コスト | 低 | 高（チームスキル要考慮） |

**判断の原則**: パブリックAPI・マイクロサービス間通信 → REST。BFF（Backend for Frontend）・モバイル複数クライアント → GraphQL検討。

---

### バージョニング戦略

- **URLパス方式** (`/v1/users`) — 最もシンプル、キャッシュしやすい。**推奨**
- **ヘッダー方式** (`API-Version: 2026-01`) — URLをきれいに保てるが、ルーティング複雑
- **クエリパラメータ方式** (`?version=2`) — 非推奨（ログ・キャッシュに漏れる）

**廃止戦略**:
- 旧バージョンは最低6ヶ月は並行稼働
- `Sunset` ヘッダーで廃止日を通知
- 変更はSemVer的に考える（破壊的変更 = メジャーバンプ）

---

### レート制限の設計

- **単位**: ユーザーID > IPアドレス（IPはNAT共有で誤検知しやすい）
- **アルゴリズム**:
  - Token Bucket → バーストを許容しつつ平均を制限（APIに最適）
  - Fixed Window → 実装が簡単だが境界でスパイク可能
- **応答**: `429 Too Many Requests` + `Retry-After` ヘッダー必須
- **設計値の例**: 無料ユーザー 100req/min、有料 1000req/min

---

### 認証設計（Firebase Auth + FastAPI）

- Firebase Auth でトークン発行 → バックエンドで `verify_id_token()` 検証
- **認証 ≠ 認可**: トークン検証後に「このユーザーはこのリソースにアクセス可能か」を別途チェック
- サービス間通信: Firebase Auth ではなく IAM サービスアカウント + OIDC トークン

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| `GET /getUsers` | `GET /users` — 動詞をURLに入れない |
| 全フィールドを毎回返す | 必要なフィールドのみ返す（過剰取得を避ける） |
| エラーを全部200で返す | 適切なHTTPステータスコード（400/401/403/404/429/500） |
| バージョンなし公開API | 最初から `/v1/` を付ける |
| レート制限なし | 必ずレート制限 + `Retry-After` を実装 |
| エラーにスタックトレースを含める | 本番ではエラーIDのみ返し、ログに詳細を記録 |

---

## コード例（FastAPI + Firebase Auth + レート制限）

```python
from fastapi import FastAPI, Depends, HTTPException, Request
from firebase_admin import auth
import time
from collections import defaultdict

app = FastAPI()
rate_store = defaultdict(list)  # 本番はRedisを使う

def rate_limit(user_id: str, limit: int = 100, window: int = 60):
    now = time.time()
    timestamps = [t for t in rate_store[user_id] if now - t < window]
    if len(timestamps) >= limit:
        raise HTTPException(status_code=429, headers={"Retry-After": "60"})
    rate_store[user_id] = timestamps + [now]

async def get_current_user(request: Request):
    token = request.headers.get("Authorization", "").removeprefix("Bearer ")
    try:
        decoded = auth.verify_id_token(token)
        rate_limit(decoded["uid"])
        return decoded
    except Exception:
        raise HTTPException(status_code=401, detail="Unauthorized")

@app.get("/v1/users/me")
async def get_me(user=Depends(get_current_user)):
    return {"uid": user["uid"], "email": user.get("email")}
```

---

## トレードオフ

| 選択 | メリット | デメリット |
|---|---|---|
| REST | シンプル、HTTP標準で動く | エンドポイント増加、over/under fetch |
| GraphQL | フレキシブル、型安全 | 複雑なキャッシュ、N+1リスク |
| URLバージョニング | 明快、キャッシュ容易 | URLが変わる、旧バージョン保守コスト |
| Token Bucketレート制限 | バースト許容で UX良い | 実装やや複雑（Redisが必要） |
| Firebase Authに依存 | 実装コスト低 | ベンダーロックイン、カスタム認証が難しい |

---

## チェックリスト

- [ ] URLに動詞を入れていない（RESTful設計）
- [ ] 最初から `/v1/` プレフィックスをつけている
- [ ] `429 Too Many Requests` + `Retry-After` を返すレート制限がある
- [ ] エラーレスポンスに本番でスタックトレースを含めていない
- [ ] 認証（誰か）と認可（何ができるか）を分離して実装している
