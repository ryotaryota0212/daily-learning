# API設計のベストプラクティス：REST vs GraphQL、バージョニング、レート制限、認証

## 概要

API設計はシステム全体の「契約」であり、一度公開すると変更コストが高い。AI時代においても、APIの設計品質がシステムの拡張性・保守性・セキュリティを決定する。「とりあえず動くエンドポイント」ではなく、「壊れにくく、変更に強く、悪用されにくいAPI」を設計する力が求められる。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いているケース | リソース指向、キャッシュ重視、公開API | 柔軟なクエリ、複数リソース結合、モバイル最適化 |
| キャッシュ | HTTPキャッシュがそのまま使える | 難しい（POSTが多い） |
| Over-fetch/Under-fetch | 起きやすい | 解決できる |
| 学習コスト | 低い | 高い |
| セキュリティ | 標準的 | クエリ深度・複雑度制限が必要 |

**FastAPI + Neon スタックでは**：基本はREST、複雑なデータ集約が必要な場面のみGraphQL（Strawberry等）を検討。

### APIバージョニング戦略

- **URLパス方式**：`/v1/users`、`/v2/users` — 最も明確、キャッシュしやすい ✅推奨
- **ヘッダー方式**：`Accept: application/vnd.api+json;version=2` — URLが綺麗だがデバッグしにくい
- **クエリパラメータ**：`/users?version=2` — ルーティングが複雑になる ❌非推奨

---

## アンチパターン vs 正しい設計

### アンチパターン
- バージョニングなしで破壊的変更を加える → クライアント全壊
- 認証トークンをURLクエリに含める → ログに残ってセキュリティ事故
- エラーレスポンスが不統一 → クライアント実装が複雑化
- レート制限なし → DDoSやAPI乱用に無防備

### 正しい設計
- 後方互換性を維持しながら新バージョンを追加
- 認証はAuthorizationヘッダー（Bearer token）
- 統一されたエラー形式（RFC 7807 Problem Details）
- レート制限 + 適切な429レスポンス

---

## コード/設計例

```python
# FastAPI での統一エラー形式 + レート制限の例
from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import JSONResponse
import time

app = FastAPI()

# シンプルなインメモリレート制限（本番はRedis使用）
rate_store: dict = {}

def check_rate_limit(client_id: str, limit: int = 100, window: int = 60):
    now = time.time()
    key_data = rate_store.get(client_id, {"count": 0, "reset": now + window})
    if now > key_data["reset"]:
        key_data = {"count": 0, "reset": now + window}
    key_data["count"] += 1
    rate_store[client_id] = key_data
    if key_data["count"] > limit:
        raise HTTPException(
            status_code=429,
            detail={"type": "rate_limit_exceeded", "retry_after": int(key_data["reset"] - now)}
        )

# RFC 7807 形式のエラーレスポンス
@app.exception_handler(HTTPException)
async def http_exception_handler(request: Request, exc: HTTPException):
    return JSONResponse(
        status_code=exc.status_code,
        content={"type": f"error/{exc.status_code}", "detail": exc.detail}
    )
```

---

## トレードオフ

| 設計選択 | メリット | デメリット |
|----------|----------|------------|
| URLバージョニング | 明確、キャッシュしやすい | URL冗長、古いバージョンの保守コスト |
| GraphQL | 柔軟なクエリ | N+1問題、キャッシュ難、攻撃面が増える |
| 厳格なレート制限 | 安全 | 正規ユーザーも弾く可能性 |
| JWT認証（ステートレス） | スケールしやすい | トークン失効が即座にできない |

**Cloud Run での注意点**：インスタンスが複数起動するため、インメモリのレート制限は機能しない。必ずRedis（Memorystore）等の共有ストアを使う。

---

## チェックリスト

- [ ] バージョニング戦略を最初に決め、URLパス方式（`/v1/`）を採用しているか
- [ ] 認証トークンはAuthorizationヘッダーのみで受け取り、URLには含めていないか
- [ ] エラーレスポンスが統一形式（type, detail, status）になっているか
- [ ] レート制限がCloud Run全インスタンス共有のストア（Redis等）で実装されているか
- [ ] 破壊的変更（フィールド削除、型変更）は新バージョンでのみ行っているか
