# API設計のベストプラクティス（REST vs GraphQL、バージョニング、レート制限、認証）

## 概要

APIはシステムの「契約」であり、一度公開すると変更コストが高い。設計の質が、長期的な保守性・スケーラビリティ・セキュリティを左右する。
AI時代においても「どのAPI構造が壊れにくく拡張しやすいか」を判断できる力は不変の価値を持つ。
FastAPI + Cloud Run + Firebase Auth スタックでの具体的な実践を前提に解説する。

---

## 仕組みの要点

### REST vs GraphQL の判断基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いている用途 | CRUD中心、単純なリソース操作 | 複雑な関連データ、柔軟なフェッチ |
| キャッシュ | HTTPキャッシュがそのまま使える | クエリごとに異なるためキャッシュ難 |
| 型安全性 | OpenAPIで補完 | スキーマ定義が強力 |
| N+1問題 | URL設計で回避可能 | DataLoaderが必要 |
| 採用基準 | **公開APIやモバイルアプリ** | **BFF・社内ダッシュボード** |

**判断ルール**: 外部公開 or シンプルなCRUD → REST。フロントが複数・関連データが複雑 → GraphQL。迷ったらREST。

### バージョニング戦略

- **URLパス方式**（推奨）: `/v1/users`, `/v2/users` → シンプル、キャッシュしやすい
- **ヘッダー方式**: `API-Version: 2` → URLが綺麗だがテストしにくい
- **非推奨**: クエリパラメータ方式（`?version=2`） → キャッシュが効かない
- **バージョン互換性の維持**: 既存フィールドの削除・型変更は禁止。追加のみ許容

### レート制限設計

- **対象の粒度**: IPベース < ユーザーIDベース < APIキーベース（精度順）
- **アルゴリズム**: トークンバケット（バースト許容）が最もユーザー体験に優しい
- **レスポンスヘッダー**: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `Retry-After` を必ず返す
- **429 Too Many Requests** + `Retry-After` でクライアントに再試行タイミングを通知

### 認証・認可の正しい設計

- **Firebase Auth JWT** をAuthorizationヘッダーに: `Bearer <token>`
- 認証（誰か）と認可（何ができるか）は分離して実装
- エンドポイントごとにスコープ/ロールを明示的に定義
- サービス間通信（Cloud Run → Cloud Run）はサービスアカウントのIDトークンを使用

---

## アンチパターン vs 正しい設計

### アンチパターン

- `GET /getUserData?action=delete` → GETで副作用を起こす
- `POST /api/v1/createNewUserAndSendEmailAndUpdateProfile` → 1エンドポイントに複数責務
- エラーレスポンスが全部 `200 OK` + body内の `"status": "error"` → HTTPセマンティクス無視
- バージョンなしで既存フィールドの型を変更 → 後方互換性の破壊
- レート制限なしの公開API → DDoS・スクレイピングへの無防備

### 正しい設計

- HTTPメソッドをセマンティクス通りに使用（GET/POST/PUT/PATCH/DELETE）
- リソース指向のURL設計: `/users/{id}/orders/{order_id}`
- エラーは適切なHTTPステータスコード + 構造化エラーボディ
- 破壊的変更は新しいバージョンで提供
- レート制限はAPI全体とエンドポイント個別の2層で設定

---

## コード/設計例（FastAPI）

```python
from fastapi import APIRouter, Depends, HTTPException, Request
from fastapi.responses import JSONResponse

router = APIRouter(prefix="/v1")

# エラーレスポンスの標準化
def error_response(code: str, message: str, status: int):
    return JSONResponse(
        status_code=status,
        content={"error": {"code": code, "message": message}}
    )

# レート制限ヘッダーをミドルウェアで付与
@app.middleware("http")
async def add_rate_limit_headers(request: Request, call_next):
    response = await call_next(request)
    response.headers["X-RateLimit-Limit"] = "100"
    response.headers["X-RateLimit-Remaining"] = str(get_remaining(request))
    return response

# リソース指向のエンドポイント
@router.get("/users/{user_id}/orders")
async def get_user_orders(user_id: str, current_user=Depends(verify_token)):
    if current_user.uid != user_id:
        raise HTTPException(status_code=403, detail="Forbidden")
    return await fetch_orders(user_id)
```

---

## トレードオフ

| 選択 | メリット | デメリット |
|------|----------|------------|
| URLバージョニング | シンプル・キャッシュしやすい | URL設計が煩雑になりうる |
| 厳格なレート制限 | 安全・コスト予測可能 | 正当なユーザーのUX低下リスク |
| GraphQL採用 | 柔軟・型安全 | キャッシュ・セキュリティが複雑化 |
| 粗い粒度のエラー | 実装簡単 | クライアント側のデバッグが困難 |

**基本方針**: REST + URLバージョニング + ユーザーIDベースのレート制限が、ほとんどのユースケースでシンプルで壊れにくい。

---

## チェックリスト

- [ ] HTTPメソッドとステータスコードをセマンティクス通りに使っているか
- [ ] バージョン管理戦略を決め、破壊的変更のポリシーを文書化しているか
- [ ] レート制限を実装し、適切なヘッダーをレスポンスに含めているか
- [ ] 認証（Firebase Auth JWT）と認可（ロール/スコープ）を分離して実装しているか
- [ ] エラーレスポンスの構造を標準化し、全エンドポイントで統一しているか
