# API設計のベストプラクティス

## 概要

AIがコードを書く時代において、「どんなAPIを設計するか」の判断力は人間の仕事として残り続ける。
REST・GraphQL・バージョニング・認証・レート制限を正しく設計できるかどうかが、
システムの拡張性・安全性・運用コストを決定づける。「とりあえず動くエンドポイント」ではなく
「壊れにくく、進化できるインターフェース」を設計する視点を持つことが重要。

---

## 仕組みの要点

### REST vs GraphQL の判断基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いているケース | リソースが明確、公開API、CDNキャッシュ重視 | 多様なクライアント、N+1問題解決、柔軟なクエリ |
| キャッシュ | GETは自然にキャッシュ可能 | クエリが動的でキャッシュ難 |
| 型安全 | OpenAPIで補完 | スキーマが型定義を兼ねる |
| 学習コスト | 低い | サーバー/クライアント双方に高い |

**判断の原則:**
- 公開API・モバイル向け・CDN必須 → REST
- 複数クライアントが異なるフィールドを要求する → GraphQL
- チームがGraphQL未経験 → RESTで始めてBFF層で補完

### バージョニング戦略

- **URLパス方式**: `/v1/users` → シンプルで最も一般的。CloudRunのトラフィック分割と相性良
- **ヘッダー方式**: `API-Version: 2024-01-01` → URLが変わらないがテスト・共有が難しい
- **非推奨**: クエリパラメータ方式（`?version=1`）→ キャッシュキーが複雑化

**バージョン管理の原則:**
- 破壊的変更（フィールド削除・型変更）は必ず新バージョンを切る
- 旧バージョンは最低6ヶ月は維持してDeprecation通知を出す
- 追加（新フィールド）は後方互換なのでバージョン不要

---

## アンチパターン vs 正しい設計

### アンチパターン

```
# 動詞をURLに入れる（RPC風REST）
POST /api/getUser
POST /api/deleteAllUsers  # 認証なしで呼べたら終わり

# エラーを全部200で返す
{"status": "error", "code": 200, "message": "Not found"}

# バージョニングなしで破壊的変更
DELETE /api/users/{id}/address_string  # → address_objectに変更して本番クラッシュ
```

### 正しい設計

```
# リソース指向 + 適切なHTTPメソッド
GET    /v1/users/{id}
PATCH  /v1/users/{id}
DELETE /v1/users/{id}

# 適切なHTTPステータス
404  Not Found
422  Unprocessable Entity（バリデーションエラー）
429  Too Many Requests（レート制限）
503  Service Unavailable（依存サービス障害時）
```

---

## コード/設計例（FastAPI + Cloud Run想定）

```python
from fastapi import FastAPI, Depends, HTTPException, Request
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
app = FastAPI()

@app.get("/v1/users/{user_id}")
@limiter.limit("60/minute")  # レート制限
async def get_user(
    request: Request,
    user_id: str,
    current_user = Depends(verify_firebase_token),  # 認証
):
    if current_user.uid != user_id:
        raise HTTPException(status_code=403)
    return await fetch_user(user_id)

# エラーレスポンスの統一形式
class ErrorResponse(BaseModel):
    error: str        # machine-readable code
    message: str      # human-readable
    request_id: str   # トレース用
```

---

## トレードオフ

| 設計判断 | メリット | コスト |
|---------|---------|--------|
| 厳密なバージョニング | 破壊的変更を安全に展開 | 旧バージョンの維持コスト |
| GraphQL採用 | クライアント自律でフィールド取得 | サーバー複雑度・キャッシュ難 |
| レート制限をAPI GW側で実装 | アプリコード簡潔 | Cloud Armorなど追加コスト |
| 認証をミドルウェア統一 | 漏れが起きにくい | 例外ルートの管理が必要 |

**FastAPI + Cloud Runスタックでの推奨:**
- レート制限: `slowapi` or Cloud Armor（WAF統合時）
- 認証: FirebaseのIDトークン検証をDependsで統一
- バージョニング: URLパス方式（`/v1/`, `/v2/`）+ Cloud Runトラフィック分割で段階移行

---

## チェックリスト

- [ ] 全エンドポイントに認証Dependsが付いているか（未認証エンドポイントは明示的に除外）
- [ ] 破壊的変更は `/v2/` など新バージョンに切り出しているか
- [ ] レート制限が設定されており、429レスポンスに `Retry-After` ヘッダーがあるか
- [ ] エラーレスポンスが統一フォーマットで `request_id` を含むか
- [ ] 公開APIにはOpenAPI（`/docs`）が整備されており、破壊的変更の影響範囲を把握できるか
