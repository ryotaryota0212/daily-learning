# API設計のベストプラクティス（REST vs GraphQL・バージョニング・認証）

## 概要

APIはシステムの「契約」であり、一度公開すると変更コストが跳ね上がる。
AI時代においても「どの設計が壊れにくく・スケールしやすく・理解しやすいか」を判断する能力は不変。
FastAPI + Firebase Auth + Cloud Run スタックで実装する際のベストプラクティスをまとめる。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 適したユースケース | CRUD中心・シンプルなリソース | 複雑なリレーション・多様なクライアント |
| キャッシュ | URLベースで容易（CDN対応○） | クエリごとに異なり難しい |
| 型安全 | OpenAPIで補完 | スキーマが強制 |
| 学習コスト | 低い | 高い |
| Over-fetch問題 | 発生しやすい | 解決できる |

**判断基準**: モバイル+Webで同一APIを使い、データ要求が多様 → GraphQL。それ以外 → REST。

### RESTの設計原則

- **リソース指向**: `/users/{id}/orders` のように名詞でリソースを表現
- **HTTPメソッドを意味通りに使う**: GET（取得）、POST（作成）、PUT（置換）、PATCH（部分更新）、DELETE（削除）
- **ステータスコードを適切に**: 200/201/204/400/401/403/404/409/429/500
- **べき等性**: PUT・DELETE は何度実行しても同じ結果になるよう設計

### バージョニング戦略

- **URLパス**: `/v1/users` → 最も明示的。推奨
- **ヘッダー**: `Accept: application/vnd.api+json;version=1`  → URLをクリーンに保てるが複雑
- **クエリパラメータ**: `/users?version=1` → 避けるべき（キャッシュが汚染される）

**原則**: v1は永遠にサポートを続ける前提で設計する。廃止する場合は最低6ヶ月前にDeprecation-Headerで通知。

### 認証フロー（Firebase Auth + FastAPI）

```python
from fastapi import Depends, HTTPException
from firebase_admin import auth

async def verify_token(authorization: str = Header(...)):
    token = authorization.removeprefix("Bearer ")
    try:
        decoded = auth.verify_id_token(token)
        return decoded
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")

@app.get("/v1/me")
async def get_me(user=Depends(verify_token)):
    return {"uid": user["uid"]}
```

---

## アンチパターン vs 正しい設計

### アンチパターン

```
# 動詞をURLに含める（RPC的）
GET /getUsers
POST /createUser
POST /deleteUser?id=123

# エラー情報を隠す
{"error": true}  # 何のエラーか不明

# 認証をクエリパラメータで渡す
GET /users?token=abc123  # ログに残る
```

### 正しい設計

```
# リソース指向
GET  /v1/users
POST /v1/users
DELETE /v1/users/{id}

# エラーレスポンスを統一
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "email is required",
    "details": [{"field": "email", "issue": "required"}]
  }
}

# 認証はAuthorizationヘッダー
Authorization: Bearer <token>
```

---

## レート制限の設計

- **単位**: ユーザーUID単位 > IPアドレス単位（共有NATで誤爆しにくい）
- **ヘッダーで状態を返す**:
  - `X-RateLimit-Limit: 100`
  - `X-RateLimit-Remaining: 42`
  - `X-RateLimit-Reset: 1722384000`
- **超過時は429 Too Many Requests** + `Retry-After` ヘッダー
- **Cloud Run + Upstash Redis** でスライディングウィンドウ実装が現実的

---

## トレードオフ

| 設計選択 | メリット | デメリット |
|----------|----------|------------|
| URLバージョニング | 明示的・キャッシュしやすい | URLが長くなる |
| 厳格なHTTPメソッド | 予測可能・ドキュメント不要 | 操作によっては不自然 |
| 詳細なエラーレスポンス | デバッグしやすい | 情報漏洩リスク（本番では詳細を隠す） |
| GraphQL | Over-fetch解消 | N+1問題・キャッシュ複雑 |
| べき等性の保証 | リトライ安全 | 実装コスト増（冪等キー管理等） |

---

## チェックリスト

- [ ] エラーレスポンスのフォーマットが全エンドポイントで統一されている
- [ ] 認証トークンをURLやクエリパラメータに含めていない
- [ ] レート制限を実装し、超過時に429とRetry-Afterを返している
- [ ] APIバージョンをURLに含め、古いバージョンの廃止計画がある
- [ ] OpenAPIスキーマ（Swagger）が自動生成されており、常に最新状態を保っている
