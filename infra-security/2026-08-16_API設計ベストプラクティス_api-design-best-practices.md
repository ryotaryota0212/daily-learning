# API設計のベストプラクティス — REST vs GraphQL・バージョニング・レート制限・認証

## 概要

APIはシステムの「契約」であり、一度公開すると変更コストが高い。設計の誤りは後から修正しにくく、クライアント側に破壊的変更を強いる。AI時代においても「どういうAPIにするか」を決める判断力はエンジニアに求められる核心スキルだ。FastAPI + Cloud Run スタックで実務に即した設計判断ができることを目指す。

## 仕組みの要点

### REST vs GraphQL 選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向き | リソース単位の操作が明確 | 複雑な関係データを柔軟に取得 |
| Over/Under fetch | 発生しやすい | クライアントが必要なフィールドを指定 |
| キャッシュ | HTTP キャッシュが使いやすい | 工夫が必要（GET+クエリハッシュ等） |
| 学習コスト | 低い | スキーマ定義・リゾルバ設計が必要 |
| 適合 | 本スタック（FastAPI）の基本 | BFFや複雑なUI駆動データ取得 |

**判断基準**: クライアントが1種類でリソース操作が明確 → REST。複数クライアント（Web/モバイル）が異なるデータ形状を要求する → GraphQL検討。

### APIバージョニング戦略

- **URLパス**: `/v1/users` — 最も明確。互換性管理がしやすい（推奨）
- **ヘッダー**: `Accept: application/vnd.api.v2+json` — URLを汚さないが発見しにくい
- **クエリパラメータ**: `?version=2` — 簡単だがキャッシュと相性悪い

**運用原則**:
- メジャーバージョンのみURLに含める（`/v1`、`/v2`）
- 旧バージョンは最低6ヶ月は維持してから廃止
- `Sunset` ヘッダーで廃止予定日を通知

### レート制限の実装層

1. **Cloudflare（エッジ）**: DDoS・ボット対策。IPベースで最初に弾く
2. **Cloud Run（アプリ）**: ユーザーIDベースの細かい制御
3. **DB（Neon）**: コネクション数制限。PgBouncer で保護

レスポンスヘッダーに状態を返す:
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1724000000
```
超過時は `429 Too Many Requests` + `Retry-After` を返す。

### 認証設計（Firebase Auth + FastAPI）

- Firebase Auth でトークン発行（クライアント責務）
- FastAPI で `Authorization: Bearer <ID_token>` を検証
- Cloud Run の `--no-allow-unauthenticated` + IAM でサービス間認証も追加

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|--------------|----------|
| `POST /getUser` — 動詞をパスに入れる | `GET /users/{id}` — HTTPメソッドで操作を表す |
| エラー時も `200 OK` を返す | 適切なHTTPステータスコードを使う |
| バージョンなしで破壊的変更 | `/v1`を維持し`/v2`を追加する |
| エラーレスポンスに実装詳細を含める | `{ "error": "code", "message": "..." }` に統一 |
| レート制限なし | ユーザー/IP単位でレート制限を設ける |

## コード/設計例（FastAPI）

```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import HTTPBearer
import firebase_admin
from firebase_admin import auth

app = FastAPI()
security = HTTPBearer()

async def verify_token(token = Depends(security)):
    try:
        decoded = auth.verify_id_token(token.credentials)
        return decoded
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")

# v1 ルーター
from fastapi import APIRouter
v1 = APIRouter(prefix="/v1")

@v1.get("/users/{user_id}")
async def get_user(user_id: str, user=Depends(verify_token)):
    if user["uid"] != user_id:
        raise HTTPException(status_code=403, detail="Forbidden")
    return {"id": user_id, "email": user["email"]}

app.include_router(v1)
```

レート制限（Redis を使う場合のイメージ）:
```python
async def rate_limit(user=Depends(verify_token), redis=Depends(get_redis)):
    key = f"rate:{user['uid']}"
    count = await redis.incr(key)
    if count == 1:
        await redis.expire(key, 60)  # 1分ウィンドウ
    if count > 100:
        raise HTTPException(status_code=429, detail="Too many requests")
```

## トレードオフ

| 選択 | メリット | デメリット |
|------|---------|----------|
| URLバージョニング | 明確・キャッシュ可 | URLが増える |
| 厳格なレート制限 | 安定性・公平性 | 正当なバーストを弾く可能性 |
| GraphQL導入 | 柔軟なデータ取得 | スキーマ管理・N+1問題・キャッシュ複雑化 |
| 認証をアプリ層で完結 | 細かい制御 | Cloud Runのエッジ認証と二重管理になる |

**推奨**: 本スタックでは REST + URLバージョニング + Cloudflareレート制限 + Firebase Auth で十分。GraphQLは「複数クライアントが異なるデータ形状を要求する」具体的な課題が出てから検討。

## チェックリスト

- [ ] HTTPメソッドとステータスコードを正しく使っているか（`GET`/`POST`/`PUT`/`DELETE`、`200`/`201`/`400`/`401`/`404`/`429`）
- [ ] `/v1` プレフィックスを設け、破壊的変更は新バージョンで行っているか
- [ ] レート制限を Cloudflare（IPベース）+ アプリ（ユーザーIDベース）の2層で設けているか
- [ ] Firebase Auth のトークン検証を全エンドポイントに適用しているか
- [ ] エラーレスポンスの形式を統一し、内部実装を露出させていないか
