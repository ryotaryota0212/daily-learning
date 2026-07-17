# API設計のベストプラクティス：REST vs GraphQL、バージョニング、レート制限、認証

## 概要

API設計は「今動けばいい」で終わらない。拡張性・後方互換性・セキュリティ・パフォーマンスを最初から考慮しないと、後で全部作り直すことになる。AI時代でもAPIの設計力は差別化要因であり、「どの構造が一番シンプルで壊れにくいか」を選べることが重要。

---

## REST vs GraphQL の判断基準

### REST を選ぶとき
- リソース中心の操作（CRUD）が明確
- キャッシュ（HTTP/CDN）を活用したい
- クライアントが複数（Web/Mobile/外部）で仕様固定
- チームにGraphQL知識がない

### GraphQL を選ぶとき
- クライアントごとに必要なフィールドが異なる（オーバーフェッチ解消）
- 複数リソースを1リクエストにまとめたい（アンダーフェッチ解消）
- フロントエンドが主導で型を決めたい
- **注意**: N+1問題（DataLoaderが必須）、キャッシュが複雑、認証設計が難しい

### FastAPI + Cloud Run スタックでの推奨
- **基本はREST**。BFFパターン（Backend for Frontend）でクライアント別エンドポイントを用意する方が運用コストが低い
- GraphQLは「本当にフロントエンドのクエリ自由度が必要」になってから導入する

---

## バージョニング戦略

### アンチパターン
```
# URLにv1を埋め込んで後で詰む
GET /v1/users/123  # v2で構造変更 → クライアント全部対応が必要
# バージョンなしで壊す
GET /users/123     # レスポンス構造を変えて既存クライアントを壊す
```

### 正しい設計
```
# URLバージョニング（シンプルで推奨）
GET /v1/users/123
GET /v2/users/123  # v1を一定期間並行稼働してから廃止

# ヘッダーバージョニング（URLを汚さない）
GET /users/123
Accept: application/vnd.myapp.v2+json
```

**原則**:
- `v1` は最低6ヶ月はサポートしてから廃止
- Deprecation-Date ヘッダーをレスポンスに含めて事前通知
- Breaking change（フィールド削除、型変更）はバージョンアップ必須
- 追加フィールドは後方互換（クライアントは無視すればいい）

---

## レート制限の設計

### アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| レート制限なしで公開 | エンドポイント別に制限を設定 |
| 全ユーザー同一制限 | 認証済み vs 未認証で差をつける |
| 超過時に500を返す | 429 Too Many Requests + Retry-After ヘッダー |
| IPのみで制限 | ユーザーID + IPの組み合わせ |

### FastAPI での実装例
```python
from fastapi import HTTPException, Request
import time

# シンプルなインメモリ実装（本番はRedisを使う）
request_counts: dict[str, list[float]] = {}

def rate_limit(user_id: str, limit: int = 100, window: int = 60):
    now = time.time()
    counts = [t for t in request_counts.get(user_id, []) if now - t < window]
    if len(counts) >= limit:
        raise HTTPException(
            status_code=429,
            detail="Rate limit exceeded",
            headers={"Retry-After": str(window)}
        )
    request_counts[user_id] = counts + [now]
```

**本番設計**:
- Redis + Sliding Window アルゴリズム
- Cloud Run の場合は Cloud Armor でL7レート制限も併用
- エンドポイント別制限（`/auth/login` は厳しく、`/health` は無制限）

---

## 認証・認可のAPI設計

### Firebase Auth + FastAPI パターン
```python
from firebase_admin import auth
from fastapi import Depends, HTTPException
from fastapi.security import HTTPBearer

security = HTTPBearer()

async def get_current_user(token = Depends(security)):
    try:
        decoded = auth.verify_id_token(token.credentials)
        return decoded
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")

@app.get("/protected")
async def protected_route(user = Depends(get_current_user)):
    return {"uid": user["uid"]}
```

**認証設計の原則**:
- すべての状態変更エンドポイント（POST/PUT/DELETE）は認証必須
- `GET /health` や公開リソースは認証不要だが明示的に設計する
- スコープ（権限スコープ）をトークンに含めてエンドポイント別に検証
- サービス間通信はFirebase Auth ではなく Cloud Run のサービスアカウントを使う

---

## レスポンス設計のトレードオフ

### 一貫性のある構造
```json
// 成功
{"data": {...}, "meta": {"total": 100, "page": 1}}

// エラー（RFC 7807 Problem Details）
{"type": "validation_error", "title": "Invalid input", "status": 400, "detail": "email is required"}
```

### トレードオフ
| 選択 | メリット | デメリット |
|---|---|---|
| 全フィールド返す | シンプル | 帯域・レイテンシ増 |
| フィールド選択（?fields=） | 効率的 | キャッシュ複雑化 |
| ページネーション（offset） | 実装簡単 | 大規模で遅い |
| カーソルページネーション | 高速・一貫 | 実装コスト高 |

---

## チェックリスト

- [ ] レート制限が認証済み/未認証で分かれており、429 + Retry-After を返しているか
- [ ] バージョニング戦略があり、Breaking change の廃止スケジュールが決まっているか
- [ ] 全状態変更エンドポイントで認証・認可を検証しているか
- [ ] エラーレスポンスが一貫した構造（RFC 7807）になっているか
- [ ] GraphQL を選ぶ理由が「オーバーフェッチ/アンダーフェッチの解消」であり、N+1対策（DataLoader）が入っているか
