# API設計のベストプラクティス — REST vs GraphQL、バージョニング、レート制限

## 概要

APIは「サービスとクライアントの契約」。一度公開したAPIの破壊的変更はクライアント障害に直結する。
設計の良し悪しがスケール・保守性・セキュリティすべてに影響するため、AI時代でも最重要設計スキルの一つ。
FastAPI + Cloud Run 構成でも、原則を外すと後から修正コストが爆発する。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いている用途 | CRUD中心・シンプルなリソース操作 | 複雑なリレーション・フロント主導の柔軟取得 |
| キャッシュ | HTTPキャッシュそのまま使える | クエリ単位のため複雑 |
| 学習コスト | 低い | 高い |
| 過剰取得/不足取得 | 発生しやすい | 解決しやすい |
| **判断基準** | クライアントが1-2種類なら REST で十分 | BFF不要でフロントが多様なら GraphQL |

### バージョニング戦略

- **URLバージョン（推奨）**: `/api/v1/users` — シンプル、CDNキャッシュしやすい
- **ヘッダーバージョン**: `Accept: application/vnd.myapp.v2+json` — URLを汚さないが発見しづらい
- **非推奨化フロー**: v1→v2共存 → Sunset ヘッダーで廃止告知 → 半年後に v1 削除

### レート制限の設計

- **単位**: ユーザーID > IPアドレス（IPはNAT環境で誤爆する）
- **アルゴリズム**: Token Bucket（バーストを許可）が UX 良好
- **ヘッダーで通知**: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `Retry-After`
- **Cloud Run** では Redis (Memorystore) で分散カウンターを持つ

### 認証フロー（Firebase Auth + FastAPI）

- Firebase ID Token を `Authorization: Bearer <token>` で送信
- FastAPI 側で `firebase_admin.auth.verify_id_token()` で検証
- Cloud Run サービス間は Service Account + OIDC トークン

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---------------|-----------|
| `GET /getUser?id=1` | `GET /users/{id}` |
| 200 で全エラーを返す | 4xx/5xx を適切に使い分ける |
| v1 を突然廃止 | Sunset ヘッダーで事前告知 |
| 全フィールドを常に返す | `?fields=id,name` のスパースフィールドセット |
| エラーボディなし | `{"error": "code", "message": "..."}` 統一形式 |

---

## コード例（FastAPI + レート制限 + Firebase Auth）

```python
from fastapi import Depends, HTTPException, Request
import firebase_admin.auth as fb_auth
from redis.asyncio import Redis

# レート制限（Token Bucket 簡易実装）
async def rate_limit(request: Request, redis: Redis = Depends(get_redis)):
    uid = request.state.uid  # Firebase UID
    key = f"rate:{uid}"
    count = await redis.incr(key)
    if count == 1:
        await redis.expire(key, 60)  # 1分ウィンドウ
    if count > 100:
        raise HTTPException(429, detail="Rate limit exceeded",
                            headers={"Retry-After": "60"})

# 認証デコレータ
async def verify_token(request: Request):
    token = request.headers.get("Authorization", "").removeprefix("Bearer ")
    try:
        decoded = fb_auth.verify_id_token(token)
        request.state.uid = decoded["uid"]
    except Exception:
        raise HTTPException(401, "Invalid token")

@app.get("/api/v1/users/me", dependencies=[Depends(verify_token), Depends(rate_limit)])
async def get_me(request: Request):
    return {"uid": request.state.uid}
```

---

## トレードオフ

- **URLバージョン vs ヘッダーバージョン**: URL は直感的だがパス設計が煩雑になる。ヘッダーはクリーンだがデバッグしにくい
- **レート制限の粒度**: 細かいほど公平だが実装・運用コスト増。まず「ユーザー単位/分」から始める
- **REST vs GraphQL**: GraphQL は柔軟だが N+1 問題・認証ロジックの複雑化・キャッシュ無効化リスクがある。モノリス初期は REST 推奨
- **エラーの詳細度**: 詳細すぎるとセキュリティリスク（情報漏洩）、少なすぎるとデバッグ不能

---

## チェックリスト

- [ ] URL に動詞でなくリソース名を使っている（`/users` not `/getUsers`）
- [ ] バージョン戦略を決めており、廃止フローが定義されている
- [ ] レート制限が実装され、`429` と `Retry-After` を返している
- [ ] 全エンドポイントで認証・認可が適切に適用されている
- [ ] エラーレスポンスのフォーマットが統一されている
