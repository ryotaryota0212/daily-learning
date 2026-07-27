# API設計のベストプラクティス — REST vs GraphQL・バージョニング・レート制限・認証

## 概要

APIはシステムの「契約」であり、一度公開したら変更コストが高い。AI時代においても、API設計の品質がシステム全体の保守性・スケーラビリティを左右する。FastAPI + Firebase Auth + Cloud Run スタックを前提に、「壊れにくいAPI」を設計するための判断基準を整理する。

---

## REST vs GraphQL — 判断基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いているケース | CRUD中心・キャッシュ重要・公開API | 複雑な関係・フロント主導・BFF層 |
| N+1問題 | URLレベルで制御しやすい | DataLoaderが必須 |
| バージョニング | URLまたはヘッダで管理 | スキーマの進化（追加は後方互換） |
| 学習コスト | 低い | 高い（クライアント側も） |
| FastAPIとの相性 | ◎ネイティブサポート | △ Strawberryなど追加ライブラリ必要 |

**結論：迷ったらREST。** チームが小さい・公開API・キャッシュが重要な場合はRESTを選ぶ。BFF（Backend for Frontend）パターンやモバイルで通信量を最小化したい場合はGraphQLを検討する。

---

## バージョニング戦略

### アンチパターン
- バージョン管理なし → 破壊的変更がクライアントを壊す
- `/api/v1/` だけ用意して後回し → v2移行時に大混乱
- クエリパラメータでバージョン管理 (`?version=2`) → キャッシュが効かない

### 正しい設計
```
# URLパスバージョニング（推奨）
GET /api/v1/users/{id}
GET /api/v2/users/{id}

# ヘッダバージョニング（公開APIに向く）
GET /api/users/{id}
Accept: application/vnd.myapp.v2+json
```

- **後方互換を守る**: フィールド追加はOK、削除・改名はNG
- **非推奨期間を設ける**: v1を即廃止しない（最低6ヶ月）
- **Deprecation-Headerで通知**:
  `Deprecation: true` / `Sunset: Sat, 01 Jan 2027 00:00:00 GMT`

---

## レート制限の設計

### 基本の考え方
- **目的**: DoS防御 + 公平なリソース分配
- **単位**: ユーザーID > APIキー > IPアドレス（なりすましに強い順）
- **アルゴリズム**: Token Bucket（バースト許容）が実用的

```python
# FastAPI + Redis でシンプルなレート制限
from fastapi import Request, HTTPException
import redis.asyncio as redis

r = redis.Redis()

async def rate_limit(request: Request, limit: int = 60, window: int = 60):
    key = f"rate:{request.state.uid}"
    count = await r.incr(key)
    if count == 1:
        await r.expire(key, window)
    if count > limit:
        raise HTTPException(429, "Too Many Requests")
```

- **レスポンスヘッダで状態を返す**:
  - `X-RateLimit-Limit: 60`
  - `X-RateLimit-Remaining: 42`
  - `Retry-After: 30`（429時）

---

## 認証設計（Firebase Auth + FastAPI）

### トークン検証の正しい実装

```python
from firebase_admin import auth
from fastapi import Depends, HTTPException, Header

async def get_current_user(authorization: str = Header(...)):
    if not authorization.startswith("Bearer "):
        raise HTTPException(401)
    token = authorization[7:]
    try:
        decoded = auth.verify_id_token(token)
        return decoded
    except Exception:
        raise HTTPException(401, "Invalid token")
```

### 認証 vs 認可の分離
- **認証**: Firebaseが担う（誰か？）
- **認可**: アプリ側 + Neon RLS が担う（何ができるか？）
- サービス間通信（Cloud Run → Cloud Run）は**Google-signed JWT**を使う（Firebase tokenではない）

---

## エラーレスポンスの統一

### アンチパターン
- エラーごとに構造が違う（`{"error": "..."}` vs `{"message": "..."}` が混在）
- スタックトレースをそのまま返す（情報漏洩）

### 正しい設計（RFC 7807 / Problem Details）
```json
{
  "type": "https://example.com/errors/validation-error",
  "title": "Validation Error",
  "status": 422,
  "detail": "email フィールドが不正です",
  "instance": "/api/v1/users"
}
```

---

## トレードオフ

| 判断 | 選択肢A | 選択肢B |
|------|---------|---------|
| バージョニング方式 | URLパス（シンプル・キャッシュ可） | Headerベース（URL汚染なし・キャッシュ難） |
| レート制限の粒度 | ユーザー単位（正確） | IP単位（未認証リクエストに有効） |
| レスポンス形式 | 独自JSON | RFC 7807（標準準拠・クライアント側実装が楽） |

---

## チェックリスト

- [ ] `/api/v1/` のようにバージョンをURLに含めている
- [ ] フィールド追加は後方互換、削除時は Deprecation ヘッダを設定している
- [ ] レート制限を実装し、429時に `Retry-After` を返している
- [ ] Firebase IDトークンの検証をミドルウェアで一元化している
- [ ] エラーレスポンスの構造を全エンドポイントで統一している
