# API設計のベストプラクティス（REST vs GraphQL・バージョニング・レート制限・認証）

## 概要

APIはシステムの「契約」であり、一度公開すると変更コストが高い。設計の善し悪しがスケーラビリティ・保守性・セキュリティのすべてに影響する。AI時代においても、APIの設計力はシステムを成立させる根幹スキルであり続ける。FastAPI + Cloud Run + Firebase Auth スタックにおける実践的な設計指針を整理する。

---

## REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いているケース | リソース単位の操作が明確な場合 | クライアントごとに必要フィールドが異なる場合 |
| パフォーマンス | N+1問題が起きやすい | over-fetch/under-fetch を解消しやすい |
| キャッシュ | HTTP標準キャッシュが使いやすい | クエリ単位で難しい |
| 学習コスト | 低い | スキーマ・リゾルバ設計が必要 |
| セキュリティ | エンドポイント単位でレート制限しやすい | 複雑なクエリによるDoS対策が別途必要 |

**判断基準**
- モバイル/BFF層でクライアントの柔軟性が必要 → GraphQL
- サーバー間通信・単純なCRUD・チームが小さい → REST
- FastAPI + Cloud Run の構成なら REST から始めるのが現実的

---

## バージョニング戦略

**アンチパターン**
- バージョン管理なしで破壊的変更を加える
- クエリパラメータでバージョン管理（`?v=2`）→ ルーティングが複雑化
- 全エンドポイントを一度に移行しようとする

**正しい設計**
- URLパスバージョニングを基本とする（`/v1/users`、`/v2/users`）
- v1 を廃止する前に移行期間（最低3ヶ月）を設ける
- 後方互換な変更（フィールド追加）はバージョンを上げない
- レスポンスに `Deprecation` ヘッダーを付与して警告

```python
# FastAPI でのバージョニング例
from fastapi import APIRouter

v1_router = APIRouter(prefix="/v1")
v2_router = APIRouter(prefix="/v2")

@v1_router.get("/users/{id}")
async def get_user_v1(id: str):
    return {"id": id, "name": "..."}  # 旧レスポンス形式

@v2_router.get("/users/{id}")
async def get_user_v2(id: str, response: Response):
    response.headers["X-API-Version"] = "2"
    return {"id": id, "display_name": "...", "avatar_url": "..."}
```

---

## レート制限の設計

**実装レイヤーの選択**

```
クライアント → Cloudflare (DDoS/IP制限) → Cloud Run (アプリレベル制限)
```

- **Cloudflare**: IPベースの粗い制限（悪意あるトラフィックの遮断）
- **アプリレベル**: ユーザー/APIキー単位の細かい制限

**FastAPI でのレート制限パターン**

```python
from fastapi import Request, HTTPException
import time, collections

# シンプルなスライディングウィンドウ（本番はRedis推奨）
_rate_store: dict[str, collections.deque] = {}

def check_rate_limit(user_id: str, limit: int = 100, window: int = 60):
    now = time.time()
    bucket = _rate_store.setdefault(user_id, collections.deque())
    while bucket and bucket[0] < now - window:
        bucket.popleft()
    if len(bucket) >= limit:
        raise HTTPException(429, detail="Rate limit exceeded")
    bucket.append(now)
```

**レスポンスヘッダーで状態を通知**
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1719100800
Retry-After: 30  # 429時のみ
```

---

## 認証・認可の設計

**Firebase Auth + FastAPI の正しい実装**

```python
from fastapi import Depends, HTTPException, Security
from fastapi.security import HTTPBearer
import firebase_admin.auth as fb_auth

security = HTTPBearer()

async def get_current_user(token=Security(security)):
    try:
        decoded = fb_auth.verify_id_token(token.credentials)
        return decoded  # uid, email, custom claims が含まれる
    except Exception:
        raise HTTPException(401, "Invalid token")

@router.get("/me")
async def get_me(user=Depends(get_current_user)):
    return {"uid": user["uid"]}
```

**認可レイヤーの分離**
- 認証（誰か）: Firebase Auth トークン検証
- 認可（何ができるか）: Custom Claims または DB の役割テーブルで制御
- リソースアクセス制御: Neon の RLS で DB層でも二重に防御

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|--------------|-----------|
| `GET /deleteUser?id=1` | `DELETE /users/{id}` (HTTPメソッドをセマンティクス通りに使う) |
| エラーを全て `200 OK` で返す | 適切なHTTPステータスコード（400/401/403/404/429/500）を使う |
| レスポンスに内部実装詳細を含める | スタックトレースは本番で非表示。エラーコードのみ返す |
| 認証なしの管理エンドポイント | 全エンドポイントにデフォルトで認証を要求 |
| ページネーションなしの一覧API | `limit`/`offset` または cursor-based pagination を必須にする |

---

## トレードオフまとめ

- **REST の単純さ vs GraphQL の柔軟性**: チームが小さい・スピードが重要な初期は REST。クライアント種別が増えてきたら GraphQL を検討
- **URLバージョニング vs ヘッダーバージョニング**: URLの方が可視性が高くキャッシュしやすいが、URL設計が複雑になる
- **アプリレベルのレート制限 vs Cloudflare**: Cloudflare は安価だがアプリ固有のロジック（ユーザー別制限）は実装不可
- **厳格な認可チェック vs 開発速度**: 初期から RLS + Custom Claims の二重防御を入れるとバグ混入を防げる

---

## チェックリスト

- [ ] HTTPメソッドとステータスコードをセマンティクス通りに使っているか
- [ ] APIバージョニング戦略を決め、廃止ポリシーを明文化しているか
- [ ] レート制限をCloudflare（IPレベル）とアプリ（ユーザーレベル）の両層で実装しているか
- [ ] Firebase Auth トークン検証を全エンドポイントで統一した Dependency として実装しているか
- [ ] エラーレスポンスに内部実装の詳細（スタックトレース・DB構造）が漏れていないか
