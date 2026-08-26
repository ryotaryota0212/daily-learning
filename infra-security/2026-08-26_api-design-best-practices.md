# API設計のベストプラクティス：REST vs GraphQL・バージョニング・認証

## 概要

APIはシステムの「契約」であり、一度公開すると変更コストが高い。
設計の失敗は後からじわじわ技術的負債として効いてくる。
FastAPI + Firebase Auth + Cloud Run 構成において「壊れにくいAPI設計」を確立する知識を整理する。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| クライアント数 | 少ない・均一 | 多い・多様（モバイル/Web/外部） |
| データ取得効率 | over-fetch/under-fetch が起きやすい | 必要なフィールドだけ取れる |
| キャッシュ | HTTPキャッシュが使いやすい | 複雑（Persisted Queryが必要） |
| 学習コスト | 低い | 高い（スキーマ設計が必要） |
| 向いているユース | CRUD中心のシンプルAPI | BFF・複数クライアント向けAPI |

**判断指針：スタートはREST。複数クライアントで取得データが違う場合のみGraphQLを検討。**

### RESTのURL設計原則

- リソース名は**名詞・複数形**：`/users` `/orders/{id}/items`
- 動詞はHTTPメソッドで表現：GET=取得、POST=作成、PUT=全更新、PATCH=部分更新、DELETE=削除
- ネストは2階層まで：`/users/{id}/orders` → 3階層は設計の見直しサイン
- フィルタ・ページネーションはクエリパラメータ：`/users?role=admin&page=2&limit=20`

### バージョニング戦略

3つのアプローチとトレードオフ：

```
# URLパス（最も明確、推奨）
GET /v1/users
GET /v2/users  ← v2で破壊的変更

# ヘッダー（URLをきれいに保てる）
Accept: application/vnd.myapp.v2+json

# クエリパラメータ（手軽だが混乱しやすい）
GET /users?version=2
```

**推奨：URLパス方式。/v1/ → /v2/ の移行時は最低6ヶ月の並行稼働期間を設ける。**

### レート制限の設計

- ユーザー単位 / IPアドレス単位 / APIキー単位で制限
- レスポンスヘッダーで状態を通知：

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1724680800
```

- 超過時は `429 Too Many Requests` を返す
- Cloud Run の場合は Cloudflare WAF でエッジレイヤーにもレート制限を設置

---

## アンチパターン vs 正しい設計

### アンチパターン

```python
# NG: 動詞URL・エラー設計なし・認証チェック漏れ
@app.post("/createUser")
def create_user(data: dict):  # dictで受けるのも問題
    db.execute(f"INSERT INTO users VALUES ('{data['name']}')")  # SQLi
    return {"ok": True}  # 何が成功したか不明
```

### 正しい設計

```python
from fastapi import APIRouter, Depends, HTTPException, status
from pydantic import BaseModel, EmailStr

router = APIRouter(prefix="/v1/users", tags=["users"])

class UserCreate(BaseModel):
    name: str
    email: EmailStr

@router.post("", status_code=status.HTTP_201_CREATED)
async def create_user(
    body: UserCreate,
    current_user=Depends(require_firebase_auth),  # 認証必須
    db=Depends(get_db)
):
    user = await db.users.create(name=body.name, email=body.email)
    return {"id": user.id, "name": user.name}  # 不要なフィールドを返さない
```

---

## エラーレスポンス設計

統一されたエラー形式を決める（RFC 7807 準拠推奨）：

```json
{
  "type": "https://example.com/errors/validation",
  "title": "Validation Error",
  "status": 422,
  "detail": "email フィールドは必須です",
  "instance": "/v1/users"
}
```

| HTTPステータス | 用途 |
|--------------|------|
| 400 | クライアントのリクエスト形式ミス |
| 401 | 未認証（Authorizationヘッダーなし） |
| 403 | 認証済みだが権限なし |
| 404 | リソース存在しない |
| 409 | 競合（重複登録等） |
| 422 | バリデーションエラー |
| 429 | レート制限超過 |
| 500 | サーバー内部エラー（詳細は隠す） |

---

## Firebase Auth との認証統合

```python
async def require_firebase_auth(
    authorization: str = Header(None),
    firebase_app=Depends(get_firebase_app)
) -> dict:
    if not authorization or not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Unauthorized")
    token = authorization.split(" ")[1]
    try:
        decoded = auth.verify_id_token(token, app=firebase_app)
        return decoded  # uid, email 等が含まれる
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")
```

---

## トレードオフ

| 設計選択 | メリット | デメリット |
|---------|---------|----------|
| URLバージョニング | 明確・キャッシュしやすい | URL長くなる |
| ヘッダーバージョニング | URLきれい | テストしにくい・見落としやすい |
| 厳格なバリデーション | バグ早期発見 | 初期実装コスト |
| GraphQL採用 | 柔軟な取得 | N+1問題・キャッシュ複雑 |
| レート制限なし | 実装コストゼロ | 悪用・過負荷リスク |

---

## チェックリスト

- [ ] エンドポイントは名詞・複数形のリソース名になっているか
- [ ] 認証（Firebase Auth）は全エンドポイントで漏れなく適用されているか
- [ ] エラーレスポンスは統一フォーマットで、内部実装を露出していないか
- [ ] バージョン戦略が決まっており、破壊的変更の移行計画があるか
- [ ] レート制限がCloudflare or アプリ層で設定されているか
