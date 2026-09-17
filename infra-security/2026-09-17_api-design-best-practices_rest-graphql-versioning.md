# API設計のベストプラクティス（REST vs GraphQL・バージョニング・レート制限・認証）

## 概要

APIはシステム間の契約であり、一度公開すると変更コストが高い。  
設計の悪いAPIは、クライアント側の修正・下位互換性の維持・セキュリティパッチのたびに全体へ影響を波及させる。  
FastAPI + Firebase Auth + Cloud Run スタックで「壊れにくく・拡張しやすく・安全な」API設計の要点を押さえる。

---

## REST vs GraphQL の判断基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いているケース | リソース中心・キャッシュ重視・複数クライアント種別 | フロントが多様なデータ形状を必要とする・N+1問題回避 |
| キャッシュ | CDN・HTTPキャッシュが効く | クエリ単位でキャッシュ困難（Persisted Queryで改善） |
| セキュリティ | メソッド・パス単位でIAM制御しやすい | クエリの深さ・複雑度制限が必要 |
| 学習コスト | 低い | スキーマ設計・リゾルバ実装のコスト大 |

**結論：** 小〜中規模のSaaSはREST一択。GraphQLはBFF（Backend for Frontend）層に限定するのが安全。

---

## バージョニング戦略

- **URLパス方式**（推奨）：`/v1/users` → 明示的・CDNキャッシュが効く
- **ヘッダ方式**：`API-Version: 2024-01` → URLが綺麗だがキャッシュ困難
- **クエリパラメータ**：`?version=2` → 非推奨（ログ・URL汚染リスク）

バージョン廃止ポリシーを最初から決める（例：6ヶ月前に Deprecation-Header を返す）。

---

## アンチパターン vs 正しい設計

### アンチパターン
- `POST /getUser`（動詞をパスに含める）
- レスポンスに `200 OK` + エラーメッセージ（ステータスコードの意味を無効化）
- 全フィールドを返す肥大レスポンス（モバイル帯域・DBクエリの無駄）
- 認証トークンをクエリパラメータに含める（ログに残る）

### 正しい設計
- リソース名は複数形の名詞：`GET /v1/users/{id}`
- HTTPステータスを正しく使う：201/400/401/403/404/409/422/500
- フィールド選択・ページネーションをサポート：`?fields=id,name&limit=20&cursor=xxx`
- 認証は `Authorization: Bearer <token>` ヘッダのみ

---

## コード例（FastAPI + Firebase Auth + レート制限）

```python
from fastapi import FastAPI, Depends, HTTPException, Request
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
app = FastAPI()

@app.get("/v1/users/{user_id}")
@limiter.limit("30/minute")
async def get_user(
    request: Request,
    user_id: str,
    current_user: dict = Depends(verify_firebase_token),
):
    # RLSはNeon側で担保。アプリ層では所有権チェックのみ
    if current_user["uid"] != user_id:
        raise HTTPException(status_code=403)
    return await db.fetch_user(user_id)
```

---

## レート制限設計

- **ユーザー単位**：認証済みは寛大（100req/min）、未認証は厳しく（10req/min）
- **エンドポイント単位**：書き込みAPIは読み込みより厳しく設定
- **レスポンスヘッダ**で残余を返す：`X-RateLimit-Remaining: 87`
- 超過時は `429 Too Many Requests` + `Retry-After` ヘッダ

---

## トレードオフ

| 選択 | メリット | デメリット |
|------|---------|-----------|
| URLパスバージョニング | 明示的・運用しやすい | URLが長くなる |
| カーソルページネーション | 大量データで安定 | オフセット指定不可 |
| レート制限をアプリ層で | 細粒度制御可能 | スケールアウト時に共有状態が必要（Redis） |
| 認証をFirebase Authに委譲 | 実装コスト低・セキュリティ高 | Firebase障害時の影響範囲が広がる |

---

## チェックリスト

- [ ] `Authorization: Bearer` ヘッダで認証し、クエリパラメータにトークンを含めていない
- [ ] 全エンドポイントにレート制限を設定し、429レスポンスに `Retry-After` を付与している
- [ ] バージョン廃止スケジュールを決め、`Deprecation` ヘッダを返すようにしている
- [ ] エラーレスポンスは `{"error": {"code": "USER_NOT_FOUND", "message": "..."}}` 形式で統一されている
- [ ] ページネーションはカーソル方式を採用し、`LIMIT OFFSET` によるパフォーマンス劣化を防いでいる
