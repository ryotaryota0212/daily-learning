# API設計のベストプラクティス（REST vs GraphQL、バージョニング、レート制限、認証）

## 概要

APIはシステムの「契約」であり、一度公開すると変更が難しい。設計ミスは後から高い代償を払う。
REST・GraphQL・認証・バージョニング・レート制限それぞれに正しい判断軸があり、「なんとなく選ぶ」ではなくトレードオフを理解した上で選択することが重要。
FastAPI + Cloud Run + Firebase Auth のスタックでは、設計決定がそのままコスト・セキュリティ・開発速度に直結する。

---

## REST vs GraphQL：判断基準

### REST を選ぶ場面
- リソースが明確で独立している（ユーザー、注文、商品）
- キャッシュが重要（CDN・HTTPキャッシュが効く）
- チームがGraphQLに不慣れ
- 外部公開API（ドキュメント・互換性管理が容易）

### GraphQL を選ぶ場面
- 1画面で複数リソースを組み合わせる（ネスト取得が多い）
- フロントが必要なフィールドだけ取りたい（Over-fetching解消）
- BFF（Backend for Frontend）パターンで内部API

### 実際の判断軸
| 観点 | REST | GraphQL |
|------|------|---------|
| キャッシュ | 容易（URL単位） | 難しい（POSTが多い） |
| N+1問題 | 起きやすい | DataLoaderで解決可能 |
| スキーマ型安全 | OpenAPI | スキーマファースト |
| 学習コスト | 低 | 高 |

---

## バージョニング戦略

### アンチパターン
```
# バージョン管理なし → 破壊的変更でクライアント全滅
GET /users

# URLパスバージョニング（多用しすぎ）
GET /v1/users  # v1のコードが永遠に残る負債
GET /v2/users  # ほぼ同じコードが二重管理
```

### 正しい設計
```
# 後方互換性を保ちつつフィールド追加
GET /users         # 常に最新。フィールド追加はOK、削除はNG
GET /v1/users      # 破壊的変更時のみバージョンを切る

# Deprecation ヘッダーで移行を促す
Deprecation: true
Sunset: Sat, 31 Dec 2026 23:59:59 GMT
Link: <https://api.example.com/v2/users>; rel="successor-version"
```

**原則：破壊的変更（フィールド削除・型変更・認証方式変更）のときだけバージョンを上げる。**

---

## レート制限の設計

### アンチパターン
- レート制限なし → DDoS・コスト爆発
- 全エンドポイントに同じ制限 → 認証エンドポイントが攻撃されやすい

### 正しい設計パターン

```python
# FastAPI + Redis でスライディングウィンドウ
from fastapi import HTTPException, Request
import redis, time

r = redis.Redis()

def rate_limit(key: str, limit: int, window: int):
    now = time.time()
    pipe = r.pipeline()
    pipe.zremrangebyscore(key, 0, now - window)
    pipe.zadd(key, {str(now): now})
    pipe.zcard(key)
    pipe.expire(key, window)
    _, _, count, _ = pipe.execute()
    if count > limit:
        raise HTTPException(429, "Rate limit exceeded")
```

### エンドポイント別制限値の例
| エンドポイント | 制限 | 理由 |
|----------------|------|------|
| `POST /auth/login` | 5回/分 | ブルートフォース防止 |
| `GET /users` | 100回/分 | 通常利用 |
| `POST /ai/generate` | 10回/分 | 高コスト操作 |

---

## 認証設計

### アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|----------------|------------|
| APIキーをURLパラメータに含める | Authorizationヘッダーに入れる |
| JWTを検証せず信頼する | 署名・有効期限・発行者を必ず検証 |
| 全エンドポイントに同じ権限 | スコープ・ロールで細分化 |
| トークン無効化ができない | Redis でブラックリスト or 短いTTL |

### FastAPI + Firebase Auth の実装パターン

```python
from firebase_admin import auth
from fastapi import Depends, HTTPException
from fastapi.security import HTTPBearer

bearer = HTTPBearer()

async def verify_token(token = Depends(bearer)):
    try:
        decoded = auth.verify_id_token(token.credentials)
        return decoded  # {"uid": "...", "email": "...", "role": "..."}
    except Exception:
        raise HTTPException(401, "Invalid token")

@app.get("/protected")
async def protected(user = Depends(verify_token)):
    return {"uid": user["uid"]}
```

---

## レスポンス設計の原則

### アンチパターン
```json
// 構造が一貫しない
{"data": [...]}  // 成功時
{"message": "error"}  // エラー時
```

### 正しい設計（RFC 7807 Problem Details）
```json
// エラー統一フォーマット
{
  "type": "https://api.example.com/errors/rate-limited",
  "title": "Too Many Requests",
  "status": 429,
  "detail": "5 requests per minute allowed",
  "retry_after": 60
}
```

---

## トレードオフまとめ

| 設計選択 | メリット | デメリット |
|----------|----------|------------|
| REST | シンプル・キャッシュ容易 | Over-fetching |
| GraphQL | 柔軟・型安全 | N+1・キャッシュ困難 |
| URLバージョニング | 明示的 | 古いバージョンの負債 |
| ヘッダーバージョニング | URLが綺麗 | テストしにくい |
| JWTステートレス | スケール容易 | 即時無効化できない |
| セッション | 即時無効化可 | 状態管理が必要 |

---

## チェックリスト

- [ ] 破壊的変更時のみバージョンを上げる運用ルールがある
- [ ] 認証エンドポイントに厳しいレート制限がある
- [ ] エラーレスポンスのフォーマットが全エンドポイントで統一されている
- [ ] Firebase Auth トークンの署名・有効期限・issuerを検証している
- [ ] `Deprecation` / `Sunset` ヘッダーで廃止予告を出す仕組みがある
