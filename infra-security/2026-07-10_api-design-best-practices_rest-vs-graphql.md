# API設計のベストプラクティス：REST vs GraphQL・バージョニング・認証

## 概要

APIはシステムの「境界面」であり、設計ミスは後から直せない技術的負債になる。
クライアントとの契約（インターフェース）を適切に定義し、スケール・セキュリティ・後方互換性を担保することが重要。
AI時代でも「どの設計を選ぶか・なぜか」を説明できる力が評価の核心になる。

---

## REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いているケース | リソース単位が明確なCRUD | クライアントごとに取得フィールドが異なる |
| N+1問題 | URLネストで対処 | DataLoaderで解決必要 |
| キャッシュ | CDN・HTTPキャッシュが効く | クエリが複雑でキャッシュしにくい |
| 学習コスト | 低い | スキーマ・リゾルバー設計が必要 |
| 推奨スタック | Cloud Run + FastAPI | BFF（Backend for Frontend）パターン |

**結論**: FastAPI + Cloud Runスタックでは、まずRESTで設計する。クライアントが複数（Web/Mobile）でフィールド要求が異なる場合のみGraphQLを検討。

---

## APIバージョニング戦略

**3つのアプローチと判断基準**

- **URLパス方式** `/v1/users` → 最もシンプル。推奨。
- **ヘッダー方式** `API-Version: 2024-01` → URLを汚さないが実装・テストが煩雑
- **クエリパラメータ** `?version=2` → キャッシュ・ログが汚れる。非推奨

**バージョンを切るタイミング**
- レスポンス構造の破壊的変更（フィールド削除・型変更）
- 認証方式の変更
- 原則: フィールド追加は後方互換のため新バージョン不要

---

## 認証設計（Firebase Auth + Cloud Run）

```python
# FastAPI + Firebase Auth の認証ミドルウェア
from firebase_admin import auth
from fastapi import HTTPException, Security
from fastapi.security import HTTPBearer

bearer = HTTPBearer()

async def verify_token(token = Security(bearer)):
    try:
        decoded = auth.verify_id_token(token.credentials)
        return decoded  # uid, email etc.
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")

@app.get("/api/v1/profile")
async def get_profile(user = Depends(verify_token)):
    return {"uid": user["uid"]}
```

- IDトークンの検証はFirebase Admin SDKに任せる（手動検証しない）
- Cloud Run → Cloud Run のサービス間通信はService Account + ID Token
- APIキー認証は管理系・M2Mのみ。ユーザー向けには使わない

---

## レート制限の設計

**実装位置の選択**

| 位置 | 特徴 |
|------|------|
| Cloudflare WAF | DDoS対策・大量リクエストの遮断 |
| APIゲートウェイ | ユーザー単位のレート制限 |
| アプリ内（slowapi等） | 細粒度制御。DB/Redis負荷に注意 |

```python
# FastAPI + slowapi によるレート制限
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@app.get("/api/v1/search")
@limiter.limit("30/minute")
async def search(request: Request):
    ...
```

- 認証済みユーザーはIPではなくUID単位で制限する
- 429レスポンスには `Retry-After` ヘッダーを必ず付与

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|--------------|-----------|
| `/getUser` `/createUser`（動詞URL） | `GET /users/{id}` `POST /users`（名詞+メソッド） |
| エラーも200で返す | 適切なHTTPステータスコード（400/401/403/404/500） |
| ページネーションなしで全件返す | `?limit=20&cursor=xxx` のカーソルページネーション |
| バージョンなしでAPI公開 | 最初から `/v1/` プレフィックスをつける |
| エラーメッセージに内部情報を含める | `detail: "Internal error"` のみ（スタックトレース非公開） |

---

## トレードオフ

- **REST の柔軟性 vs 一貫性**: URLが増えるほど設計が発散しやすい。OpenAPI Specで縛る
- **レート制限の厳しさ vs UX**: 厳しすぎると正常ユーザーが弾かれる。段階的に実装
- **バージョン維持コスト**: v1/v2を並行維持するほど保守コストが上がる。廃止計画を最初から決める
- **GraphQLの柔軟性 vs セキュリティ**: クエリの深さ制限・コスト制限がないと過負荷クエリが通る

---

## チェックリスト

- [ ] URLはリソース名詞 + HTTPメソッドで設計されているか
- [ ] `/v1/` プレフィックスが最初から付いているか
- [ ] 全エンドポイントに認証ミドルウェアが適用されているか（公開APIは明示的に除外）
- [ ] レート制限が実装され、429に `Retry-After` が含まれるか
- [ ] エラーレスポンスに内部情報（スタックトレース、DBエラー）が漏れていないか
