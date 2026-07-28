# API設計のベストプラクティス（REST vs GraphQL・バージョニング・レート制限・認証）

## 概要

AI時代においてAPIは「システムの境界面」であり、設計の良し悪しがシステム全体の品質を決める。
悪いAPI設計は後から直せず、クライアントを巻き込んだ破壊的変更を強いる。
REST・GraphQL・gRPCそれぞれの適用場面を理解し、「壊れにくく・使いやすく・スケールする」APIを設計する力が求められる。

---

## 仕組みの要点

### REST vs GraphQL vs gRPC の選択基準

| 観点 | REST | GraphQL | gRPC |
|------|------|---------|------|
| 向いている場面 | 公開API・シンプルCRUD | 複雑なUIデータ取得 | 内部マイクロサービス間通信 |
| クライアントの柔軟性 | 低（固定レスポンス）| 高（必要フィールドのみ） | 低（スキーマ固定） |
| キャッシュしやすさ | ○（URLベース）| △（POSTのため難しい） | × |
| 型安全 | △（OpenAPIで補完） | ○ | ◎（Protobuf） |
| 学習コスト | 低 | 中 | 高 |

**選択の鉄則：**
- 外部公開API → REST（理解しやすく、キャッシュ効く）
- 複雑なダッシュボード・N+1問題を解決したい → GraphQL
- サービス間通信・ストリーミング → gRPC

---

## アンチパターン vs 正しい設計

### アンチパターン
- **動詞をURLに含める**: `/api/getUser`, `/api/createPost`
- **バージョンなし**: `/api/users` → 後から変更不能
- **エラーレスポンスが一貫しない**: 場所によって `{error: "msg"}` だったり `{message: "msg"}` だったり
- **認証をクエリパラメータで渡す**: `/api/data?token=xxx` → ログに残る
- **過剰なレスポンス**: 常に全フィールドを返す（帯域・CPU無駄）

### 正しい設計
- **リソース名詞 + HTTPメソッドで操作を表現**
- **バージョニングはURLパスに含める**: `/api/v1/users`
- **エラーレスポンスを標準化**: RFC 7807 Problem Details 形式
- **認証はAuthorizationヘッダ**: `Authorization: Bearer {token}`
- **フィールドフィルタリング対応**: `?fields=id,name,email`

---

## コード/設計例（FastAPI + Firebase Auth）

```python
from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel
from typing import Optional

router = APIRouter(prefix="/api/v1")

# 標準エラーレスポンス（RFC 7807準拠）
class ProblemDetail(BaseModel):
    type: str
    title: str
    status: int
    detail: str

# リソース設計: 名詞 + HTTPメソッド
@router.get("/users/{user_id}")
async def get_user(
    user_id: str,
    fields: Optional[str] = None,  # ?fields=id,name でフィールド絞り込み
    current_user=Depends(verify_firebase_token),  # Authorizationヘッダから検証
):
    user = await db.get_user(user_id)
    if not user:
        raise HTTPException(
            status_code=404,
            detail=ProblemDetail(
                type="https://example.com/errors/not-found",
                title="User Not Found",
                status=404,
                detail=f"User {user_id} does not exist",
            ).dict(),
        )
    if fields:
        allowed = set(fields.split(","))
        return {k: v for k, v in user.items() if k in allowed}
    return user
```

---

## バージョニング戦略

| 戦略 | 例 | メリット | デメリット |
|------|-----|---------|-----------|
| URLパス | `/api/v1/users` | 明確・キャッシュしやすい | URL変更が必要 |
| ヘッダ | `API-Version: 2026-07` | URLがきれい | キャッシュしにくい |
| クエリパラメータ | `?version=1` | シンプル | 見落としやすい |

**推奨：URLパスバージョニング**
- `/api/v1/` → 破壊的変更があるときだけv2へ
- 後方互換な変更（フィールド追加）はバージョンアップ不要
- v1とv2は最低6ヶ月並行稼働させてから旧バージョン廃止

---

## レート制限の設計

```python
# Cloud Run + Redis でのスライディングウィンドウ実装イメージ
async def check_rate_limit(user_id: str, limit=100, window=60):
    key = f"rate:{user_id}:{int(time.time() // window)}"
    count = await redis.incr(key)
    if count == 1:
        await redis.expire(key, window)
    if count > limit:
        raise HTTPException(
            status_code=429,
            headers={"Retry-After": str(window), "X-RateLimit-Limit": str(limit)},
            detail="Rate limit exceeded",
        )
```

**レート制限の設計ポイント：**
- レスポンスヘッダで状態を通知: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `Retry-After`
- ユーザー単位・IPアドレス単位・APIキー単位を使い分ける
- エンドポイントごとに異なる制限（書き込み系は厳しく）

---

## トレードオフ

| 設計判断 | メリット | デメリット |
|---------|---------|-----------|
| GraphQL採用 | N+1解消・柔軟なクエリ | キャッシュ困難・認可実装が複雑化 |
| 細粒度API（1エンドポイント1操作）| シンプル・テストしやすい | クライアントがN回叩く必要が出る |
| 粗粒度API（バルク操作）| ラウンドトリップ削減 | 複雑化・エラーハンドリングが難しい |
| 厳格なバージョン管理 | 破壊的変更が安全 | 並行管理コスト |
| レスポンスに全フィールド | 実装シンプル | 過剰データ・帯域・セキュリティリスク |

**AI時代の判断軸：**
- 「クライアントを壊さずに変更できるか？」を常に問う
- 設計の複雑さよりも、変更容易性を優先する

---

## チェックリスト

- [ ] URLはリソース名詞で設計されており、動詞が含まれていない
- [ ] エラーレスポンスが全エンドポイントで統一されている（RFC 7807推奨）
- [ ] 認証トークンはAuthorizationヘッダで送受信している（URLパラメータ禁止）
- [ ] バージョニングポリシーが定義され、旧バージョンの廃止期限が明確
- [ ] レート制限が実装され、超過時にRetry-Afterヘッダで案内している
