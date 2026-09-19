# API設計のベストプラクティス（REST vs GraphQL・バージョニング・レート制限・認証）

## 概要

APIはシステムの「契約」であり、一度公開すると変更コストが非常に高い。AI時代においては、AIがAPIを呼び出すケースも増え、設計の良し悪しが開発速度・保守性・セキュリティに直結する。「とりあえず動くAPI」ではなく「壊れにくく、拡張しやすく、安全なAPI」を設計する力が求められる。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| ユースケース | CRUD・シンプルなリソース操作 | 複雑なデータ取得・BFF |
| クライアントの自由度 | 低（サーバー定義） | 高（クエリで指定） |
| キャッシュ | 容易（HTTP標準） | 難しい（POST中心） |
| N+1問題 | URLで制御 | DataLoader必須 |
| 学習コスト | 低 | 高 |

**原則**: リソースが明確で操作がシンプルならREST。複数リソースをまとめて取得したい場合や、モバイル・BFF用途はGraphQL。

### REST設計の基本

- **リソース命名**: 名詞・複数形 `/users/{id}/posts`（動詞NG: `/getUser`）
- **HTTPメソッド正しい使用**: GET=読取、POST=作成、PUT=完全更新、PATCH=部分更新、DELETE=削除
- **ステータスコード**: 200/201/204/400/401/403/404/409/422/500 を正確に返す
- **冪等性**: GET・PUT・DELETEは冪等、POST・PATCHは非冪等を意識

### バージョニング戦略

```
# URL方式（最も一般的、キャッシュしやすい）
/api/v1/users
/api/v2/users

# ヘッダー方式（URLをクリーンに保てるが、ルーティング複雑）
Accept: application/vnd.myapp.v2+json

# クエリパラメータ方式（非推奨: キャッシュ汚染リスク）
/api/users?version=2
```

**推奨**: URL方式（`/v1/`）+ 最低2バージョンを並行サポート + deprecation headerで廃止を通知。

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|----------------|-----------|
| `/getUser`, `/createPost` (動詞URL) | `/users`, `/posts` (名詞URL) |
| 全エラーに200を返す | 適切なHTTPステータスコード |
| エラー詳細を本番で露出 | `error_code` + `message`のみ返す |
| 認証なしで全APIを公開 | Firebase Auth + ミドルウェアで全保護 |
| バージョンなしでAPI公開 | 最初から `/v1/` を付ける |
| ページネーションなし | cursor/offset必須、max 100件制限 |

---

## コード例（FastAPI + Firebase Auth）

```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import HTTPBearer
from firebase_admin import auth

app = FastAPI()
security = HTTPBearer()

async def get_current_user(token = Depends(security)):
    try:
        decoded = auth.verify_id_token(token.credentials)
        return decoded
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")

@app.get("/v1/users/{user_id}")
async def get_user(user_id: str, current_user = Depends(get_current_user)):
    if current_user["uid"] != user_id:
        raise HTTPException(status_code=403, detail="Forbidden")
    return {"id": user_id, "email": current_user.get("email")}
```

### レート制限（Cloud Run + Cloudflare）

```python
# slowapi でレート制限（FastAPI）
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@app.get("/v1/search")
@limiter.limit("30/minute")
async def search(request: Request, q: str):
    return {"results": []}
```

---

## トレードオフ

| 判断 | メリット | デメリット |
|------|----------|-----------|
| URL バージョニング | シンプル・キャッシュ容易 | URL冗長、ルーティング増加 |
| GraphQL採用 | 柔軟なクエリ | N+1問題・キャッシュ困難 |
| 厳格なスキーマバリデーション | バグ早期発見 | 開発速度低下 |
| レート制限をアプリ層で実装 | 細かい制御可能 | スケール時に課題→Cloudflareへ移管推奨 |

---

## チェックリスト

- [ ] 全エンドポイントに Firebase Auth ミドルウェアが適用されているか
- [ ] URLは `/v1/` から始まり、リソース名は名詞・複数形か
- [ ] 全エラーレスポンスが適切なHTTPステータスコードを返しているか
- [ ] ページネーション（cursor or offset）と上限件数が設定されているか
- [ ] レート制限がCloudflareまたはアプリ層で設定されているか
