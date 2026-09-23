# API設計のベストプラクティス（REST vs GraphQL・バージョニング・レート制限・認証）

## 概要

APIはシステムの「契約」。一度公開すると変更コストが高い。
FastAPI + Cloud Run スタックでは、設計ミスが後から大きな技術的負債になる。
「とりあえず動くAPI」ではなく「壊れにくく、進化できるAPI」を最初から設計する力が求められる。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いているケース | CRUD中心、シンプルなリソース | 複雑なネスト、クライアント主導のデータ取得 |
| キャッシュ | HTTPキャッシュが使いやすい | クエリごとに異なるため難しい |
| 学習コスト | 低い | 高い（スキーマ設計含む） |
| N+1問題 | 個別に対処 | DataLoader等が必要 |
| **判断基準** | チームが小さい・スピード重視 | フロントエンドが多様・BFF的用途 |

> **結論**: 迷ったらREST。GraphQLはフロントエンドの要求が複雑になってから導入する。

### バージョニング戦略

- **URLパス方式**: `/v1/users` → 最もシンプルで推奨
- **ヘッダー方式**: `Accept: application/vnd.myapi.v1+json` → 綺麗だが煩雑
- **クエリパラメータ**: `?version=1` → 非推奨（キャッシュ汚染）

```
推奨パターン:
GET /v1/users       ← 安定版
GET /v2/users       ← 新バージョン（v1は非推奨期間後に削除）
```

### レート制限の設計

- 制限単位: IP単位 < ユーザー単位 < APIキー単位（精度順）
- ヘッダーで状態を返す: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `Retry-After`
- 超過時は `429 Too Many Requests`

### 認証フロー（Firebase Auth + FastAPI）

```
1. クライアント → Firebase Auth でIDトークン取得
2. クライアント → Authorization: Bearer <IDトークン>
3. FastAPI → firebase_admin.auth.verify_id_token() で検証
4. 検証済みUID をリクエストコンテキストに付与
```

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---------------|-----------|
| `/getUser`, `/deleteUser` のような動詞URL | `GET /users/{id}`, `DELETE /users/{id}` |
| エラー時も `200 OK` を返す | 適切なHTTPステータスコードを使う |
| バージョン管理なし | `/v1/` プレフィックスから始める |
| 全エンドポイントが認証不要 | デフォルト認証必須・例外だけ公開 |
| エラーにスタックトレースを含める | 本番では汎用メッセージのみ返す |

---

## コード例（FastAPI + Firebase Auth 認証ミドルウェア）

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import firebase_admin.auth as fb_auth

security = HTTPBearer()

async def get_current_user(
    creds: HTTPAuthorizationCredentials = Depends(security)
):
    try:
        decoded = fb_auth.verify_id_token(creds.credentials)
        return decoded  # {"uid": "...", "email": "..."}
    except Exception:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED)

@app.get("/v1/me")
async def get_me(user=Depends(get_current_user)):
    return {"uid": user["uid"]}
```

---

## トレードオフ

- **URLバージョニング**: シンプルだが、URL増殖・重複実装のリスク
- **GraphQL**: 柔軟性↑だが、キャッシュ・セキュリティ・学習コストが上がる
- **強い認証**: セキュリティ↑だが、開発・テストが面倒になる
- **細かいレート制限**: 攻撃耐性↑だが、実装・運用コスト増

---

## チェックリスト

- [ ] エンドポイントはリソース名（名詞）＋HTTPメソッドで設計されているか
- [ ] `/v1/` プレフィックスを付けてバージョン管理しているか
- [ ] 全認証必須エンドポイントで `Depends(get_current_user)` を使っているか
- [ ] エラーレスポンスに内部情報（スタックトレース）を含めていないか
- [ ] レート制限のヘッダー（`X-RateLimit-*`）をクライアントに返しているか
