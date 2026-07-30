# API設計のベストプラクティス（REST vs GraphQL・バージョニング・レート制限・認証）

## 概要

APIはシステムの「契約」である。設計の甘さは後から修正が極めて難しく、破壊的変更によってクライアントを壊す。
FastAPI + Firebase Auth + Cloud Run スタックでは、APIの設計品質がそのままサービスの堅牢性・保守性に直結する。
「動くエンドポイント」ではなく「長期間維持できる契約」を設計できることがAI時代のエンジニアの価値になる。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|---|---|---|
| クライアント多様性 | 少ない（同一チームのみ）→ REST | 多い（Web/iOS/Android）→ GraphQL |
| 取得データの可変性 | 固定的 → REST | 柔軟に必要 → GraphQL |
| キャッシュ | URLベースで容易 | 難しい（POSTのみ） |
| エラーハンドリング | HTTP statusコードが自然 | 常に200、bodyでエラー表現 |
| チームの習熟度 | 高い | 低ければコスト大 |

**判断基準**: まずRESTで始め、クライアント間でN+1や過剰フェッチが頻発したらGraphQLを検討する。

### バージョニング戦略

- **URL versioning**: `/api/v1/users` → 最もシンプル、推奨
- **Header versioning**: `Accept: application/vnd.api+json;version=1` → URLがクリーンだが発見しにくい
- **Query param**: `/users?version=1` → キャッシュが効きにくい

**原則**: 破壊的変更（フィールド削除・型変更）は必ずバージョンを上げる。追加のみなら不要。

### レート制限の設計

- ユーザーIDベース（認証済み）と IPベース（未認証）を分けて設定
- `429 Too Many Requests` + `Retry-After` ヘッダーを返す
- レート制限情報をレスポンスヘッダーで公開: `X-RateLimit-Limit`, `X-RateLimit-Remaining`

### 認証フロー（Firebase Auth + FastAPI）

1. クライアントが Firebase で認証 → IDトークン取得
2. `Authorization: Bearer <IDトークン>` でAPIリクエスト
3. Cloud Run サービスがトークンを検証 → `uid` を信頼できるユーザー識別子として使用

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| `/getUser`, `/createUser`, `/deleteUser` (動詞URL) | `GET /users/{id}`, `POST /users`, `DELETE /users/{id}` |
| エラーを全て `200 OK` で返す | `400`, `401`, `403`, `404`, `422`, `500` を適切に使う |
| v1 → v2 移行なしにフィールドを削除 | Deprecation通知期間を設け段階的に移行 |
| 全エンドポイントに同じレート制限 | 重い処理（検索・一括）は低く、軽い取得は高く設定 |
| JWTを検証せずにuidを信頼 | 毎リクエストでFirebase Admin SDKで検証 |

---

## コード例（FastAPI + Firebase Auth）

```python
from fastapi import Depends, HTTPException, Header
from firebase_admin import auth

async def verify_token(authorization: str = Header(...)) -> dict:
    token = authorization.removeprefix("Bearer ")
    try:
        return auth.verify_id_token(token)
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")

@app.get("/api/v1/users/me")
async def get_me(claims: dict = Depends(verify_token)):
    uid = claims["uid"]
    # uidを使ってNeonからユーザー情報取得
    return {"uid": uid}

# レート制限ヘッダーをレスポンスに付与
@app.middleware("http")
async def add_rate_limit_headers(request, call_next):
    response = await call_next(request)
    response.headers["X-RateLimit-Limit"] = "100"
    return response
```

---

## トレードオフ

- **厳格なバージョニング** → 後方互換性は高いが、複数バージョン維持のコスト増
- **レート制限を厳しくする** → 安全性は高いが、正当なバースト利用を阻害するリスク
- **GraphQL** → 柔軟性は高いが、N+1問題・キャッシュ困難・権限管理の複雑化
- **全エンドポイントをv1に含める** → 移行コストを先送りできるが、将来の変更が難しくなる

---

## チェックリスト

- [ ] HTTPメソッドとURL設計がRESTの原則に従っているか（名詞URL・適切なメソッド）
- [ ] 全エンドポイントでFirebase IDトークンを検証しているか（uid信頼しない）
- [ ] 破壊的変更時にAPIバージョンを上げる運用ルールがあるか
- [ ] レート制限が認証済み/未認証ユーザーで異なる設定になっているか
- [ ] エラーレスポンスにHTTPステータスコードと構造化メッセージが含まれているか
