# API設計のベストプラクティス — REST vs GraphQL・バージョニング・レート制限・認証

## 概要

APIはシステムの「公開インターフェース」であり、一度公開すると変更コストが高い。
設計の誤りがセキュリティ脆弱性・スケーラビリティ問題・クライアント負債に直結する。
FastAPI + Cloud Run 構成では特にバージョニングと認証設計が長期運用を左右する。
AI時代においても「どう使われるか」を先読みした設計力が差別化要因になる。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| ユースケース | リソースCRUD、単純な取得 | 複雑な関連、モバイル最適化 |
| N+1問題 | サーバー側で解決 | DataLoader必須 |
| キャッシュ | HTTPキャッシュそのまま使える | クエリが動的でCDNキャッシュ困難 |
| 型安全 | OpenAPI/Swagger | スキーマファースト |
| 学習コスト | 低い | 高い（DataLoader・認可の複雑さ） |

**結論：** 小〜中規模のサービスはRESTで始める。GraphQLはフロントエンドが複数・データ形状が多様なときのみ検討。

### バージョニング戦略

- **URLパス方式**（推奨）: `/v1/users`, `/v2/users`
  - 明示的、CDNキャッシュしやすい
- **ヘッダー方式**: `API-Version: 2026-01-01`
  - URLを汚さないが、テストしにくい
- **クエリパラメータ**: `?version=2`
  - 非推奨（ロギング・キャッシュが複雑）

**廃止ポリシー**:
- 新バージョン公開後 最低6ヶ月は旧バージョンを維持
- Deprecation-Date ヘッダーで事前通知

### レート制限の設計

- **スコープ**: ユーザーID > API Key > IP（精度の高い順）
- **アルゴリズム**:
  - Token Bucket: バーストを許容しながら平均レートを制御（推奨）
  - Fixed Window: 実装容易だが境界で2倍バーストが起きる
- **レスポンス**: `429 Too Many Requests` + `Retry-After` ヘッダー
- **ヘッダー**:
  ```
  X-RateLimit-Limit: 100
  X-RateLimit-Remaining: 42
  X-RateLimit-Reset: 1721649600
  ```

### 認証設計

- **エンドポイント認証**: Firebase Auth の JWT を `Authorization: Bearer <token>` で受け取る
- **サービス間認証**: Cloud Run では OIDC トークン（Google ID Token）を使う
- **APIキー管理**: キーはハッシュして保存、プレーンテキストで保存しない

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|--------------|-----------|
| `/getUsers`, `/createUser`（動詞URL） | `GET /users`, `POST /users`（名詞+HTTPメソッド） |
| 全エラーを `200 OK` で返す | 意味のあるステータスコード（400/401/403/404/429/500） |
| バージョンなしで破壊的変更 | URLバージョニング + 廃止ポリシー |
| レート制限なし | Token Bucket + ユーザースコープ |
| エラーに内部スタックトレースを含める | エラーIDのみ返し、詳細はログに記録 |
| 全フィールドを常に返す | フィールドの取捨選択（`?fields=id,name`）またはGraphQL |

---

## コード例（FastAPI + Firebase Auth）

```python
from fastapi import FastAPI, Depends, HTTPException, Header
from firebase_admin import auth as firebase_auth

app = FastAPI()

async def verify_token(authorization: str = Header(...)):
    scheme, _, token = authorization.partition(" ")
    if scheme.lower() != "bearer":
        raise HTTPException(401, "Invalid auth scheme")
    try:
        return firebase_auth.verify_id_token(token)
    except Exception:
        raise HTTPException(401, "Invalid token")

# バージョニング: プレフィックスで分離
v1 = FastAPI()
v2 = FastAPI()

@v1.get("/users/me")
async def get_me_v1(claims=Depends(verify_token)):
    return {"uid": claims["uid"]}

app.mount("/v1", v1)
app.mount("/v2", v2)
```

---

## トレードオフ

| 選択 | メリット | デメリット |
|------|---------|-----------|
| URLバージョニング | 明示的・キャッシュ容易 | URLが増える |
| GraphQL採用 | 柔軟なクエリ | N+1・認可・キャッシュが複雑 |
| 厳格なレート制限 | 保護が強い | 正当なユーザーへの影響 |
| 過去バージョン長期維持 | 後方互換性 | 保守コスト増 |

---

## チェックリスト

- [ ] エンドポイントは名詞+HTTPメソッドで設計されているか
- [ ] `/v1/` のURLプレフィックスでバージョニングされているか
- [ ] 全エンドポイントにレート制限（Token Bucket）が設定されているか
- [ ] 認証エラーは `401`、認可エラーは `403` で区別されているか
- [ ] エラーレスポンスにスタックトレースが含まれていないか
