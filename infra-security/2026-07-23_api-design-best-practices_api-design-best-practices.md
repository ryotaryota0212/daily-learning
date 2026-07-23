# API設計のベストプラクティス（REST vs GraphQL・バージョニング・レート制限・認証）

## 概要
APIはシステムの「契約」であり、一度公開したら簡単に変更できない。AI時代においても、外部連携・フロント-バック分離・マイクロサービス間通信の要となる。「とりあえず動くエンドポイント」を量産するのではなく、スケール・変更容易性・セキュリティを最初から組み込んだ設計が求められる。FastAPI + Firebase Auth + Cloud Run スタックでの実践を中心に整理する。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いているケース | リソース中心・シンプルな CRUD | フロントが多様・over-fetch が問題 |
| キャッシュ | HTTP キャッシュそのまま使える | クエリごとに異なり難しい |
| 型安全 | OpenAPI スキーマで補完 | スキーマが型定義そのもの |
| 学習コスト | 低い | 高い（N+1問題、DataLoader 等） |
| 推奨判断 | **まずこちら**。チームが小さいなら REST | フロントチームが複数 or BFF が必要なとき |

FastAPI + Neon スタックでは **REST + OpenAPI** が最もシンプルで壊れにくい。

### バージョニング戦略

- **URLパス方式**（`/v1/`, `/v2/`）：最も明示的で分かりやすい。推奨
- **ヘッダー方式**（`API-Version: 2`）：URLを汚さないが、ルーティングが複雑になる
- **クエリパラメータ方式**（`?version=2`）：非推奨。キャッシュが効きにくい

```
GET /api/v1/users/123
GET /api/v2/users/123  ← 破壊的変更があった場合のみ v2 を切る
```

**原則**：後方互換を保てる変更（フィールド追加）は同じバージョンで対応。削除・型変更は新バージョンを切る。

### レート制限の設計

```python
# FastAPI + Redis によるスライディングウィンドウ実装例
from fastapi import Request, HTTPException
import redis, time

r = redis.Redis()

def rate_limit(user_id: str, limit: int = 100, window: int = 60):
    key = f"rate:{user_id}:{int(time.time()) // window}"
    count = r.incr(key)
    r.expire(key, window)
    if count > limit:
        raise HTTPException(status_code=429, detail="Rate limit exceeded")
```

- レスポンスに `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` ヘッダーを付与
- **ユーザー単位** と **IP単位** を組み合わせる（認証前は IP、認証後はユーザーID）
- Cloud Run の場合、Cloudflare WAF と組み合わせて 2 層防御にする

### 認証の実装パターン（Firebase Auth + FastAPI）

```python
from fastapi import Depends, HTTPException, Header
from firebase_admin import auth

async def get_current_user(authorization: str = Header(...)):
    if not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401)
    token = authorization[7:]
    try:
        decoded = auth.verify_id_token(token)
        return decoded  # uid, email, role など含む
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")
```

- Firebase ID Token は **1時間**で期限切れ。フロントでリフレッシュ処理が必須
- `custom claims` で role を埋め込み、バックエンドで RBAC を実装する
- サービス間通信（Cloud Run 同士）は Firebase Token でなく **Google ID Token** を使う

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|----------------|-----------|
| `/getUser`, `/createUser` など動詞エンドポイント | `GET /users/{id}`, `POST /users` のリソース+HTTPメソッド |
| エラーを全部 `200 OK` で返す | 適切な HTTP ステータス（400/401/403/404/422/500）を使う |
| レスポンスに内部実装を露出（DB列名そのまま） | DTOで変換して公開インターフェースを安定させる |
| バージョニングなしで破壊的変更 | `/v1` → `/v2` を切り、移行期間を設ける |
| 全エンドポイントに認証なし | デフォルト認証必須、明示的に公開エンドポイントを宣言 |

---

## 設計例：FastAPI でのエンドポイント構成

```python
from fastapi import APIRouter, Depends

router = APIRouter(prefix="/api/v1", tags=["users"])

@router.get("/users/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: str,
    current_user: dict = Depends(get_current_user)
):
    # current_user["uid"] で認可チェック
    ...

@router.post("/users", response_model=UserResponse, status_code=201)
async def create_user(...):
    ...
```

---

## トレードオフ

- **バージョニング**：URLパス方式はコードが増えるが明示的。ヘッダー方式は API 利用者が混乱しやすい
- **GraphQL**：型安全・柔軟だが N+1 問題・キャッシュ複雑化・学習コストが高い。小チームでは負債になりやすい
- **レート制限**：Redisを使うとインフラ依存が増えるが精度は高い。Cloud Run のスケールアウト時にはRedisのシングルポイント障害にも注意
- **認証**：全エンドポイントに認証を課すと DX が落ちるが、セキュリティは上がる。ヘルスチェックや公開エンドポイントは明示的に除外する

---

## チェックリスト

- [ ] エンドポイントはリソース + HTTP メソッドで設計されているか
- [ ] 破壊的変更時に `/v2` への切り替え計画があるか
- [ ] レート制限は認証前（IP）・認証後（ユーザーID）の両方で設定されているか
- [ ] Firebase ID Token の検証ミドルウェアが全保護エンドポイントに適用されているか
- [ ] エラーレスポンスは適切な HTTP ステータスと構造化されたメッセージを返しているか
