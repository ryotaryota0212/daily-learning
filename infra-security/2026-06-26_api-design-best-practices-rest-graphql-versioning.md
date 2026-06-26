# API設計のベストプラクティス（REST / GraphQL / バージョニング / 認証）

## 概要

APIはシステムの「契約」であり、一度公開すると変更コストが高い。  
AI時代においても「どのAPIをどう設計するか」の判断はエンジニアの核心スキル。  
動くAPIより「壊れにくく・スケールし・運用できるAPI」を設計する視点が重要。

---

## REST vs GraphQL：判断基準

| 観点 | REST | GraphQL |
|------|------|---------|
| ユースケース | リソース操作、シンプルなCRUD | 複雑なデータ取得、フロントエンド主導 |
| キャッシュ | HTTPキャッシュが使える（GETは自然にキャッシュ） | クエリ都度異なるためキャッシュ難しい |
| 過不足フェッチ | Over/Under fetchが起きやすい | 必要なフィールドのみ取得できる |
| 学習コスト | 低い | 高い（スキーマ定義、Resolver設計） |
| 向いている場面 | パブリックAPI、マイクロサービス間通信 | BFF層、モバイル/Web統合 |

**結論：迷ったらREST。GraphQLはフロントが複数あり、over-fetchが問題になるとき。**

---

## バージョニング戦略

- **URLパス方式**（推奨）: `/api/v1/users` → 明示的で分かりやすい
- **ヘッダー方式**: `Accept: application/vnd.api+v2` → URLはきれいだが発見しにくい
- **クエリパラメータ方式**: `/users?version=2` → 非推奨（キャッシュが効きにくい）

**破壊的変更の定義**：
- フィールドの削除・型変更・必須化 → バージョンアップ必須
- フィールド追加・オプション追加 → バージョンアップ不要（後方互換）

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|--------------|-----------|
| `GET /getUser` | `GET /users/{id}` |
| エラーを全て200で返す | 適切なHTTPステータス（404/422/500） |
| 認証トークンをURLに含める | Authorizationヘッダーを使う |
| ページネーションなしで全件返す | `limit/offset` または カーソル方式 |
| エラーメッセージに内部情報を含める | 汎用メッセージ＋エラーコードのみ |

---

## FastAPI実装例（最小限）

```python
from fastapi import FastAPI, Depends, HTTPException, Header
from typing import Optional

app = FastAPI()

# バージョンプレフィックス
v1 = APIRouter(prefix="/api/v1")

@v1.get("/users/{user_id}")
async def get_user(
    user_id: str,
    authorization: str = Header(...),  # Bearerトークン
    limit: int = 20,                   # ページネーション
    cursor: Optional[str] = None
):
    if limit > 100:
        raise HTTPException(422, detail="limit must be ≤ 100")
    # ... 実装
    return {"data": [...], "next_cursor": "abc123"}

app.include_router(v1)
```

---

## レート制限の設計

- **戦略**：ユーザーID単位 > IPアドレス単位（IPは共有NATで誤検知しやすい）
- **アルゴリズム**：Token Bucket（バーストを許容）vs Fixed Window（単純）
- **実装場所**：Cloud Run前段のCloud Armor or Cloudflareで処理するのが最善
- **レスポンスヘッダー**：`X-RateLimit-Remaining`, `Retry-After` を必ず返す

---

## 認証パターン（Firebase Auth + FastAPI）

```python
async def verify_token(authorization: str = Header(...)) -> dict:
    token = authorization.replace("Bearer ", "")
    try:
        decoded = auth.verify_id_token(token)  # Firebase Admin SDK
        return decoded
    except Exception:
        raise HTTPException(401, detail="Invalid token")
```

- サービス間通信は **IDトークン**ではなく **サービスアカウント** を使う
- 公開APIには認証不要エンドポイントを明示し、ルーティングレベルで分離

---

## トレードオフ

| 選択 | メリット | デメリット |
|------|---------|-----------|
| REST + バージョニング | シンプル・キャッシュ可 | over-fetch問題 |
| GraphQL | 柔軟なデータ取得 | N+1問題・キャッシュ難 |
| カーソルページネーション | 大量データに強い | 実装複雑・ランダムアクセス不可 |
| オフセットページネーション | 実装簡単 | 大量データで遅くなる |

---

## チェックリスト

- [ ] HTTPメソッドとステータスコードがREST規約に従っているか
- [ ] 認証トークンがヘッダー経由で渡されているか（URLに含めていないか）
- [ ] ページネーションが実装されており、上限が設定されているか
- [ ] レート制限のレスポンスヘッダーを返しているか
- [ ] 破壊的変更時にバージョンを上げるルールが合意されているか
