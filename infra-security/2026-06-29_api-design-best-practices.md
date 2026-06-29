# API設計のベストプラクティス — REST vs GraphQL、バージョニング、認証、レート制限

## 概要

APIはシステムの「公開インターフェース」であり、一度公開すると変更コストが高い。特にAI時代は複数のクライアント（Web/モバイル/LLMエージェント）が同一APIを叩くため、設計の良し悪しが運用コスト・可用性・セキュリティに直結する。「とりあえず動くAPI」ではなく「変更に強く、壊れにくいAPI」を設計する力が問われる。

---

## 仕組みの要点

### REST vs GraphQL — 選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いている用途 | リソース単位のCRUD、公開API | 複雑なデータ取得、BFF層 |
| Over/Under-fetch | 起きやすい | 解消できる |
| キャッシュ | CDNで簡単 | 工夫が必要（POST中心） |
| 学習コスト | 低い | 高め |
| スキーマ管理 | OpenAPI（任意） | 必須（型安全） |

**判断基準**: 外部公開・シンプルCRUDならREST。内部BFF・多様なクライアントがあるならGraphQL。

### バージョニング戦略

- **URLパス方式**: `/v1/users` — 最も一般的、キャッシュしやすい
- **ヘッダー方式**: `Accept: application/vnd.api+json;version=2` — URLをクリーンに保てる
- **非推奨化フロー**: 新バージョンリリース → 旧バージョンに `Deprecation` ヘッダー付与 → 6〜12ヶ月後に廃止

### 認証フロー（FastAPI + Firebase Auth）

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import firebase_admin.auth as fb_auth

security = HTTPBearer()

async def get_current_user(
    creds: HTTPAuthorizationCredentials = Depends(security)
) -> dict:
    try:
        decoded = fb_auth.verify_id_token(creds.credentials)
        return decoded
    except Exception:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED)
```

- サービス間通信は Firebase Auth ではなく **Cloud Run の OIDC トークン** を使う
- 外部APIキーはSecret Managerで管理し、コードに埋め込まない

### レート制限の設計

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@app.get("/api/search")
@limiter.limit("30/minute")  # 認証なしユーザーは厳しく
async def search(request: Request):
    ...
```

- ユーザーIDベース（認証済み）vs IPベース（未認証）で制限を分ける
- 制限超過は `429 Too Many Requests` + `Retry-After` ヘッダーで返す
- Cloudflare WAFと二重化してDDoS対策と組み合わせる

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|--------------|-----------|
| `/getUser`、`/createOrder`（動詞URL） | `/users/{id}`、`POST /orders`（リソース+HTTPメソッド） |
| エラー全部 `200 OK` + body にエラーコード | 適切なHTTPステータスコード（400/401/403/404/429/500） |
| バージョンなしで破壊的変更を本番適用 | `/v1` を維持しつつ `/v2` を並行リリース |
| 全フィールドを毎回返す（Over-fetch） | フィールドの選択（`?fields=id,name`）やGraphQL |
| 認証トークンをURLクエリパラメータに含める | `Authorization: Bearer` ヘッダーで渡す（ログに残らない） |

---

## レスポンス設計の例

```json
// 成功レスポンス（統一フォーマット）
{
  "data": { "id": "u_123", "name": "Ryota" },
  "meta": { "request_id": "req_abc", "version": "1.0" }
}

// エラーレスポンス（RFC 7807 Problem Details）
{
  "type": "https://example.com/errors/rate-limited",
  "title": "Too Many Requests",
  "status": 429,
  "detail": "Limit: 30/min. Retry after 45s.",
  "retry_after": 45
}
```

---

## トレードオフ

- **REST の一貫性 vs GraphQL の柔軟性**: REST はチーム全員が理解しやすいが、モバイル向けに最適化したい場合はGraphQLのBFFが有効
- **厳格なバージョニング vs 後方互換フィールド追加**: フィールド追加は後方互換だがスキーマが肥大化する。破壊的変更はバージョンアップ必須
- **細かいレート制限 vs 運用コスト**: ユーザーごとのレート管理はRedisが必要。シンプルにするならIPベースのみにしてCloudflare WAFに委ねる
- **詳細なエラーメッセージ vs 情報漏洩リスク**: 開発環境では詳細を返し、本番では `detail` を最小限に抑える

---

## チェックリスト

- [ ] URLはリソース名詞+HTTPメソッドで設計されているか（動詞URLを使っていないか）
- [ ] 認証トークンはヘッダーで渡し、URLやログに含まれていないか
- [ ] レート制限が実装され、429とRetry-Afterヘッダーが返るか
- [ ] 破壊的変更時にAPIバージョンをインクリメントするルールがあるか
- [ ] エラーレスポンスに内部スタックトレースや機密情報が含まれていないか
