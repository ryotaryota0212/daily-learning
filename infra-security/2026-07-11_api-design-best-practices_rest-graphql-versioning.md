# API設計のベストプラクティス

## 概要

AIがコードを書く時代でも「どんなAPIを設計するか」はエンジニアが決める。設計の良し悪しが運用コスト・スケーラビリティ・セキュリティを左右する。REST/GraphQLの選択、バージョニング戦略、レート制限、認証の組み合わせを正しく判断できることが現代のエンジニアに求められる。

---

## 仕組みの要点

### REST vs GraphQL の判断基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いているケース | リソースが独立・クライアント種別が少ない | モバイル/Webでフィールドが異なる、結合が多い |
| キャッシュ | HTTP標準のキャッシュが効く | クエリごとに変わるためキャッシュが難しい |
| 過剰/不足取得 | 発生しやすい | クライアントが必要分だけ取得できる |
| 学習コスト | 低い | スキーマ設計・N+1問題への対処が必要 |
| 監視・ログ | URLでリクエスト分類しやすい | クエリ内容で分類が必要 |

**FastAPI + Cloud Run スタックでは基本RESTを推奨。** GraphQLはBFF（Backend for Frontend）層でのみ検討する。

### バージョニング戦略

- **URLパス方式**（推奨）: `/v1/users`, `/v2/users` — 明示的でキャッシュしやすい
- **ヘッダー方式**: `Accept: application/vnd.api+json;version=2` — URLが汚れないが見えにくい
- **クエリパラメータ方式**: `?version=2` — 非推奨（ログ・キャッシュが複雑になる）

バージョン廃止ポリシー（例: 旧バージョンは新バージョンリリース後6ヶ月間サポート）を最初に決めておくことが重要。

### レート制限の設計

- **対象単位**: ユーザーID > APIキー > IPアドレスの順で優先（IPは共有NATで誤検知しやすい）
- **アルゴリズム**: Token Bucket（バースト許容）vs Fixed Window（シンプル）vs Sliding Window（精度高い）
- **ヘッダーで伝える**: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`
- **超過時**: `429 Too Many Requests` + `Retry-After` ヘッダー

### 認証フロー（Firebase Auth + FastAPI）

```
クライアント → Firebase Auth → IDトークン取得
クライアント → FastAPI (Authorization: Bearer <token>)
FastAPI → Firebase Admin SDK でトークン検証
FastAPI → uid・email をリクエストコンテキストに付与
```

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|--------------|-----------|
| エラーに全部200を返す | 適切なHTTPステータス（400/401/403/404/422/500）を使う |
| URLに動詞を入れる（`/getUser`） | リソース名+HTTPメソッド（`GET /users/{id}`） |
| バージョン管理なしで破壊的変更 | `/v2/`を切ってから段階移行 |
| レート制限なし | 最初からCloud Runのサービス設定 or ミドルウェアで実装 |
| 全エラーを同じレスポンス形式で返す | RFC 7807 Problem Details 形式を統一 |

---

## コード例（FastAPI + Firebase Auth）

```python
from fastapi import Depends, HTTPException, Header
from firebase_admin import auth

async def verify_token(authorization: str = Header(...)):
    if not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Invalid auth header")
    token = authorization.split(" ")[1]
    try:
        decoded = auth.verify_id_token(token)
        return decoded  # {"uid": "...", "email": "..."}
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")

# レート制限ミドルウェア（簡易版 - 本番はRedis推奨）
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@app.get("/v1/users/{user_id}")
@limiter.limit("60/minute")
async def get_user(user_id: str, claims=Depends(verify_token)):
    if claims["uid"] != user_id:
        raise HTTPException(status_code=403)
    return {"id": user_id}
```

---

## トレードオフ

- **URLバージョニング**: 明示的で運用しやすいが、クライアント側の修正コストが高い
- **GraphQL採用**: 柔軟性は高いがN+1問題・認可の複雑さ・キャッシュ制御が難しくなる
- **厳格なレート制限**: DDoS耐性は上がるが、正規ユーザーの体験を損なうリスクがある
- **IPベースのレート制限**: 実装が簡単だが、企業ネットワーク（共有NAT）で誤検知が多発する

---

## チェックリスト

- [ ] 全エンドポイントで適切なHTTPステータスコードを返しているか
- [ ] APIバージョン廃止ポリシーをドキュメント化しているか
- [ ] 認証トークンの検証をすべてのエンドポイントに適用しているか
- [ ] レート制限を設定し、超過時に`Retry-After`ヘッダーを返しているか
- [ ] エラーレスポンスのフォーマットがAPIを通じて統一されているか
