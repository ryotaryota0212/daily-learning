# API設計のベストプラクティス — REST vs GraphQL・バージョニング・認証

## 概要

APIはシステムの「境界線」であり、設計の良し悪しがシステム全体の拡張性・保守性に直結する。
AI時代においても「どんなAPIを公開するか」「どう認証するか」「どうバージョン管理するか」という判断は人間が担う中核スキルである。
FastAPI + Firebase Auth スタックでは設計ミスが後から直しにくいため、初期設計が特に重要。

## 仕組みの要点

### REST vs GraphQL の選択基準
- **REST**: リソース中心・HTTP標準・キャッシュしやすい・シンプルなCRUD向き
- **GraphQL**: 複雑なクエリ・必要フィールドだけ取得・複数リソースを1リクエストで取得可能
- **判断軸**: クライアント多様性が高い → GraphQL有利 / シンプルなCRUD → REST有利
- BFF（Backend for Frontend）パターン: フロントごとに最適化したAPIを用意する折衷案

### バージョニング戦略
- URLパス: `/api/v1/users` — 最も一般的・見つけやすい（**推奨**）
- ヘッダー: `Accept: application/vnd.myapp.v1+json` — URLを汚さないが発見しにくい
- クエリパラメータ: `/users?version=1` — キャッシュに影響しやすく非推奨
- **互換性破壊時のみ**バージョンを上げる。後方互換な変更はフィールド追加でOK

### 認証パターン
- Firebase Auth トークン（JWT）を `Authorization: Bearer <token>` で渡す
- Cloud Run の前段でサービスアカウントベースのIAM認証を追加
- サービス間通信は Google-managed service account で認証（トークン不要）

### レート制限の配置
- APIゲートウェイ / Cloud Run の前段で実施（Cloudflare や Cloud Armor）
- エンドポイント単位で閾値を変える（書き込み操作は厳しく、読み取りは緩く）

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| `/getUser`, `/createUser` など動詞URL | `/users/{id}` + HTTPメソッドで表現 |
| エラーを全て200で返す | 4xx/5xxを正しく使う（422 Unprocessable Entity等） |
| バージョン管理なしで破壊的変更 | `/api/v2/` を切り、v1は一定期間維持 |
| DBの列名をレスポンスにそのまま露出 | APIレイヤーでマッピングして変換（内部構造を隠す） |
| 認証なしで全エンドポイントを公開 | パブリック/プライベートを明示的に分離 |

## コード例（FastAPI + Firebase Auth）

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer
from firebase_admin import auth

security = HTTPBearer()

async def verify_token(token = Depends(security)):
    try:
        decoded = auth.verify_id_token(token.credentials)
        return decoded
    except Exception:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED)

# バージョニング: prefix で分離
app.include_router(v1_router, prefix="/api/v1")
app.include_router(v2_router, prefix="/api/v2")
```

## トレードオフ

| 観点 | REST | GraphQL |
|---|---|---|
| 学習コスト | 低 | 中〜高 |
| キャッシュ | しやすい（GETベース） | 難しい（POST主体） |
| 型安全 | OpenAPI/Pydanticで補完 | スキーマで強制 |
| Over-fetch問題 | 発生しやすい | 解決できる |
| バージョン管理 | URLで明示的に管理 | スキーマ進化で対応 |

## チェックリスト

- [ ] HTTPメソッドとステータスコードを正しく使っているか（GETは副作用なし、POSTで作成など）
- [ ] 破壊的変更時にバージョンを上げ、旧バージョンの廃止期限を明示しているか
- [ ] 全エンドポイントに「認証必要 / 不要」が明示されているか
- [ ] エラーレスポンスにスタックトレースやDB情報が含まれていないか
- [ ] レート制限がAPIゲートウェイ層（アプリの外側）で実施されているか
