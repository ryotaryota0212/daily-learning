# API設計のベストプラクティス（REST vs GraphQL・バージョニング・レート制限・認証）

## 概要

APIはシステムの「外部境界」であり、一度公開すると変更コストが高い。
設計の誤りはセキュリティ・スケーラビリティ・保守性の全てに影響する。
「とりあえず動くエンドポイント」ではなく、壊れにくく進化できるAPIを最初から設計する力が求められる。
FastAPI + Cloud Run 構成では、このドキュメントの設計原則を土台にすること。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いている用途 | リソース単位のCRUD | 複雑なネストデータの柔軟取得 |
| キャッシュ | HTTP標準キャッシュが使える | クエリ単位でキャッシュが難しい |
| 過剰取得 | Over-fetchしやすい | 必要フィールドのみ取得可能 |
| 複雑さ | シンプル | スキーマ・リゾルバの管理コスト大 |
| BFF不要で使えるか | 複数エンドポイントが増える | 1エンドポイントで柔軟に対応 |

**判断基準（FastAPI前提）:**
- 社内API・モバイルBFF → GraphQL検討
- 外部公開・マイクロサービス間 → REST
- チームが小さい・スピード優先 → REST一択

### RESTの設計原則

- URLはリソース名（名詞）で表現: `/users/{id}`, `/orders/{id}/items`
- HTTPメソッドでCRUDを表現: GET/POST/PUT/PATCH/DELETE
- ステータスコードを正しく使う: 200/201/204/400/401/403/404/409/422/500
- レスポンスに `message` だけでなく `error_code` も含める（機械処理可能に）
- ページネーションは `limit/offset` か `cursor` 方式を一貫して使う

---

## アンチパターン vs 正しい設計

### アンチパターン
```
GET /getUser?userId=123          # 動詞URLはNG
POST /updateUser                 # 操作名URLはNG
{ "status": "OK", "data": null } # 成功・失敗の表現が曖昧
```

### 正しい設計
```
GET /users/123                   # リソース名+ID
PATCH /users/123                 # 部分更新はPATCH
{ "error_code": "USER_NOT_FOUND", "message": "..." }  # 機械処理可能
```

---

## バージョニング戦略

- **URLパス方式**（推奨）: `/v1/users`, `/v2/users` → CDNキャッシュが効く
- **ヘッダー方式**: `Accept: application/vnd.api+json;version=2` → URLが汚れない
- **v1は壊さない原則**: 新フィールド追加はv1に追加可、フィールド削除・型変更はv2
- FastAPI では `APIRouter(prefix="/v1")` でバージョン分離

---

## レート制限の設計

```python
# FastAPI + Redis でのレート制限（スライディングウィンドウ）
from fastapi import Request, HTTPException
import redis, time

r = redis.Redis()

def check_rate_limit(user_id: str, limit: int = 100, window: int = 60):
    key = f"rate:{user_id}:{int(time.time()) // window}"
    count = r.incr(key)
    if count == 1:
        r.expire(key, window)
    if count > limit:
        raise HTTPException(status_code=429, detail="Rate limit exceeded")
```

- **レスポンスヘッダーに状態を返す**: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `Retry-After`
- ユーザー別 / IPアドレス別 / エンドポイント別に粒度を変える
- Cloud Run の前段に Cloud Armor でIP単位の制限も追加する

---

## 認証設計（Firebase Auth + FastAPI）

```python
from firebase_admin import auth
from fastapi import Depends, HTTPException, Header

async def get_current_user(authorization: str = Header(...)):
    token = authorization.replace("Bearer ", "")
    try:
        decoded = auth.verify_id_token(token)
        return decoded
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")
```

- Firebase ID Token は1時間で失効 → クライアントは `getIdToken(true)` で自動更新
- サービス間通信（Cloud Run同士）はFirebase AuthではなくGCP Service Accountで認証
- APIキー認証は外部パートナー向けのみ。ユーザー向けにAPIキーは使わない

---

## トレードオフ

| 選択 | メリット | デメリット |
|------|---------|---------|
| URLバージョニング | CDNキャッシュ可・明確 | URL変更の手間 |
| 厳格なレート制限 | DDoS耐性・公平性 | 正当ユーザーのUX低下 |
| GraphQL | 柔軟・型安全 | Nクエリ問題・キャッシュ難 |
| 粗いエラーメッセージ | 情報漏洩防止 | デバッグしにくい |

---

## チェックリスト

- [ ] URLはリソース名（名詞）、操作はHTTPメソッドで表現している
- [ ] バージョニング方針を決め、v1を壊さない運用ルールがある
- [ ] レート制限を実装し、`Retry-After` ヘッダーを返している
- [ ] Firebase ID Token の検証をDependency Injectionで共通化している
- [ ] エラーレスポンスに `error_code`（機械処理可能）を含めている
