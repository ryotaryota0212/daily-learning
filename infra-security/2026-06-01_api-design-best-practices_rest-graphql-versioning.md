# API設計のベストプラクティス（REST vs GraphQL、バージョニング、レート制限、認証）

## 概要

APIはシステムの「契約書」であり、一度公開すると変更コストが高い。
設計ミスは後続のクライアント全員に影響し、セキュリティホールにもなる。
「動くAPIを早く作る」ではなく「壊れにくく、変更に強いAPIを設計する」視点が重要。
特にAI時代では複数クライアント（Web/Mobile/他サービス）への一貫した提供が求められる。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 適用場面 | リソース中心、シンプルなCRUD | 複雑な関係、多様なクライアント |
| Over-fetch問題 | 起きやすい | 起きにくい |
| キャッシュ | HTTP標準で容易 | 複雑（POST固定） |
| 型安全 | OpenAPIで補完 | スキーマが強制 |
| 学習コスト | 低い | 高い |

**FastAPI + 単一フロントエンドならREST一択。** GraphQLは複数クライアントが異なるデータ形状を必要とするときに正当化される。

### APIバージョニング戦略

- **URLパス方式**（推奨）: `/api/v1/users` — 明示的で運用しやすい
- **ヘッダー方式**: `Accept: application/vnd.app.v2+json` — URLが綺麗だが発見しにくい
- **クエリパラメータ方式**: `/users?version=2` — アンチパターン

**ポイント**: v1を廃止するまでの期間（最低6ヶ月）をSLAとして定め、Deprecation警告ヘッダーを返す。

### 認証パターン（Firebase Auth + FastAPI）

- `Authorization: Bearer <Firebase IDトークン>` をリクエストヘッダーに付与
- Cloud Run側でFirebase Admin SDKを使い検証
- サービス間通信はIDトークンではなくCloud Run の **OIDC トークン**を使う

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|--------------|-----------|
| `/getUser`, `/createUser` のような動詞URL | `/users` にGET/POSTで統一 |
| 全エラーで200を返す | HTTP ステータスコードを正しく使う（400/401/403/404/500） |
| バージョニングなしで破壊的変更 | `/v2/` を作り `/v1/` に Deprecation ヘッダー |
| 認証なしのエンドポイントが混在 | 全エンドポイントをデフォルト認証必須にし、公開エンドポイントを明示的に除外 |
| レスポンスに内部エラーメッセージを露出 | `{"error": "internal_server_error"}` のみ返す |

---

## コード例（FastAPI + Firebase Auth）

```python
from fastapi import Depends, HTTPException, Security
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from firebase_admin import auth

bearer = HTTPBearer()

async def verify_token(
    cred: HTTPAuthorizationCredentials = Security(bearer)
) -> dict:
    try:
        return auth.verify_id_token(cred.credentials)
    except Exception:
        raise HTTPException(status_code=401, detail="invalid_token")

@app.get("/api/v1/users/me")
async def get_me(user: dict = Depends(verify_token)):
    return {"uid": user["uid"], "email": user.get("email")}
```

---

## レート制限設計

- **目的**: DDoS防止 + コスト保護 + フェアユース
- **実装場所**: APIゲートウェイ（Cloud Armor）かミドルウェア層
- **単位**: `user_id` または `IP` ベース、エンドポイント別に設定
- **レスポンス**: `429 Too Many Requests` + `Retry-After` ヘッダー
- **注意**: グローバルIPブロックより、ユーザー単位の制限を優先

---

## トレードオフ

| 設計選択 | メリット | デメリット |
|---------|---------|-----------|
| 厳格なバージョニング | 互換性保証、安全な廃止 | 管理コスト増、コードの重複 |
| 単一バージョン維持 | 管理シンプル | 破壊的変更が全クライアントに影響 |
| GraphQL採用 | 柔軟なクエリ | N+1問題、キャッシュ複雑化 |
| REST + 複数エンドポイント | シンプル、キャッシュ容易 | Over-fetch、エンドポイント増加 |

**FastAPI + Cloud Run構成では**: REST + URLバージョニング + Firebase Auth検証をDepends()で統一するのが最もシンプルで壊れにくい。

---

## チェックリスト

- [ ] 全エンドポイントにHTTP動詞を正しく割り当て（GET=読み取り、POST=作成、PUT/PATCH=更新、DELETE=削除）
- [ ] 認証をDependsで統一し、公開エンドポイントのみ明示的に除外している
- [ ] 破壊的変更時は `/v2/` を切り、`/v1/` に `Deprecation` ヘッダーを付与している
- [ ] エラーレスポンスに内部スタックトレースを含めていない
- [ ] レート制限の単位（ユーザー/IP）と上限値を設計書に明記している
