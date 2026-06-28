# API設計のベストプラクティス — REST vs GraphQL、バージョニング、レート制限、認証

## 概要

API設計はシステムの「契約」であり、一度公開すると変更コストが極めて高い。
AI時代においても「どの構造でAPIを公開するか」「どう壊れにくくするか」の判断はエンジニアの責任として残る。
FastAPI + Cloud Run + Firebase Auth 構成でのAPI設計の要点を整理する。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| ユースケース | CRUD中心、リソースが明確 | 複雑な関連データ、フロント主導 |
| キャッシュ | CDN・HTTPキャッシュが効く | POST多用でキャッシュ困難 |
| 学習コスト | 低い | スキーマ設計が必要 |
| 過剰取得/不足 | 発生しやすい | N+1問題に注意 |
| **結論** | SaaS・内部API・モバイルのCRUDに向く | BFF層やフロント多様性が高い場合 |

**FastAPI + Cloud Run のスタックでは REST を基本とする。** GraphQLは明確な要件がある場合のみ。

### バージョニング戦略

- **URLパス方式**（推奨）: `/v1/users`, `/v2/users` — シンプルで最も普及
- **ヘッダー方式**: `API-Version: 2` — URLがきれいだがデバッグしにくい
- **クエリパラメータ**: `/users?version=2` — キャッシュが効きにくい

**原則：v1を廃止する前に最低6ヶ月の移行期間を設ける。**

### 認証の設計

```python
# FastAPI + Firebase Auth の認証ミドルウェア
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer

bearer = HTTPBearer()

async def verify_token(token = Depends(bearer)) -> dict:
    try:
        decoded = firebase_admin.auth.verify_id_token(token.credentials)
        return decoded
    except Exception:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED)

@app.get("/v1/me")
async def get_me(user: dict = Depends(verify_token)):
    return {"uid": user["uid"]}
```

### レート制限の設計

- **アルゴリズムの選択**:
  - トークンバケット: バースト許容、API全体向き
  - スライディングウィンドウ: 正確だがメモリ消費大
  - 固定ウィンドウ: 実装簡単だが境界問題あり

- **Cloud Run でのレート制限**:
  - Cloudflare WAF or API Gateway でエッジに配置するのが基本
  - アプリ内実装は Redis（Upstash 等）で `user_id:minute` キーで管理

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|--------------|-----------|
| `/getUserById?id=1` （動詞をURLに入れる） | `GET /v1/users/1` |
| 200 OK でエラーを返す | 適切なHTTPステータスコード（400/401/404/429/500） |
| 認証なしで全エンドポイント公開 | 公開エンドポイントを明示的に列挙し、残りは認証必須 |
| バージョンなしで破壊的変更 | バージョンを上げて旧バージョンを deprecation 告知 |
| レート制限なし | ユーザーIDまたはIPでリクエスト数を制限 |

---

## エラーレスポンスの標準化

```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Retry after 60 seconds.",
    "retry_after": 60
  }
}
```

- `code`: 機械可読な識別子（クライアント側で分岐可能）
- `message`: 人間可読なメッセージ
- HTTP ヘッダーに `Retry-After`, `X-RateLimit-Remaining` を付与

---

## トレードオフ

| 決定 | メリット | デメリット |
|------|---------|-----------|
| REST を選ぶ | シンプル、CDN活用可 | 過剰取得が発生しうる |
| URLバージョニング | 明示的で管理しやすい | URL設計の複雑さ増加 |
| エッジでレート制限 | アプリ層の負荷減 | Cloudflare等の依存増加 |
| アプリ内でレート制限 | 柔軟性高い | Redis等の依存とコスト増 |

---

## チェックリスト

- [ ] 全エンドポイントに `/v1/` プレフィックスがあるか
- [ ] 認証が必要なエンドポイントは `Depends(verify_token)` が付いているか
- [ ] エラーレスポンスが `code` + `message` の統一形式か
- [ ] レート制限が Cloudflare またはアプリ層で実装されているか
- [ ] 破壊的変更を加えた際にバージョンを上げ、旧版の deprecation 日程を記載しているか
