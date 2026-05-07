# API設計のベストプラクティス — REST vs GraphQL・バージョニング・レート制限・認証

## 概要

APIはシステムの境界面であり、設計の良し悪しがスケーラビリティ・保守性・セキュリティに直結する。
AI時代でも「どのAPIを、どういう契約で公開するか」という判断はエンジニアにしか担えない。
「とりあえず動くエンドポイント」ではなく、クライアントと合意できる**安定した契約**を設計する力が問われる。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いているケース | リソースが明確・公開API | フロントが多様・モバイル最適化 |
| 学習コスト | 低い（HTTP標準） | 高い（スキーマ・リゾルバ） |
| キャッシュ | HTTPキャッシュが使いやすい | 難しい（POSTが多い） |
| 過剰取得/不足 | 起きやすい | クライアントが選択できる |
| セキュリティ | シンプル | N+1・深いクエリに注意 |

**判断の原則**: 社内BFF・モバイルが多様なら GraphQL、外部公開・単純CRUDなら REST。

### バージョニング戦略

- **URLパス方式**（`/v1/users`）: 最も明示的、キャッシュしやすい → **推奨**
- ヘッダー方式（`API-Version: 2`）: URLが綺麗だがキャッシュ複雑
- クエリパラメータ方式（`?version=2`）: テストしやすいが標準外

**ポリシー例**:
- メジャーバージョンはURLに含める（`/v1`, `/v2`）
- 廃止予定は最低6ヶ月前に `Deprecation` ヘッダーで通知
- 後方互換を破る変更 = メジャーバージョンアップ

### レート制限の設計

- **単位**: ユーザー別 / IPアドレス別 / エンドポイント別を組み合わせる
- **アルゴリズム**: Token Bucket（バースト許容）を基本とする
- **レスポンス**: `429 Too Many Requests` + `Retry-After` ヘッダー必須
- **スタック**: Cloud Run の前段に Cloudflare or API Gateway でレート制限

### 認証設計

- **サービス間**: Cloud Run → Cloud Run は Workload Identity / ID トークン
- **ユーザー認証**: Firebase Auth の JWT をヘッダーで受け取り、バックエンドで検証
- **API キー**: 公開API向け。ハッシュ化してDB保存、プレフィックスのみ表示

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---------------|-----------|
| `/getUser`, `/createUser` などRPC風URL | `/users` + HTTPメソッドで操作を表現 |
| エラーを全部200で返す | 適切な4xx/5xxステータス + error bodyを返す |
| バージョニングなしで破壊的変更 | `/v2/` に移行し古いバージョンを一定期間維持 |
| レート制限なしで公開API | ユーザー/IP単位でレート制限を必ず設ける |
| エラーメッセージにスタックトレースを含める | クライアントには汎用メッセージ、詳細はサーバーログへ |

---

## コード例（FastAPI）

```python
from fastapi import FastAPI, Depends, HTTPException, Request
from fastapi.responses import JSONResponse

app = FastAPI()

# バージョニング: ルーターで /v1 プレフィックス
from fastapi import APIRouter
v1 = APIRouter(prefix="/v1")

@v1.get("/users/{user_id}")
async def get_user(user_id: str, request: Request):
    # レート制限チェック（実際はミドルウェアorCloudflare）
    # Firebase JWT 検証は Depends で注入
    return {"id": user_id, "name": "example"}

# エラーレスポンスの統一形式
@app.exception_handler(HTTPException)
async def http_exception_handler(request, exc):
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": {"code": exc.status_code, "message": exc.detail}},
    )

app.include_router(v1)
```

---

## トレードオフ

| 選択 | メリット | デメリット |
|------|---------|-----------|
| REST | シンプル・CDNキャッシュ | 過剰取得が起きやすい |
| GraphQL | 柔軟なクエリ | 複雑・キャッシュ困難 |
| URLバージョニング | 明示的・ツールと相性◎ | URLが変わると破壊的 |
| 厳しいレート制限 | 安全 | 正規ユーザーを弾くリスク |

---

## チェックリスト

- [ ] エラーレスポンスが統一フォーマット（`{"error": {"code": ..., "message": ...}}`）になっているか
- [ ] 破壊的変更のたびにバージョンを上げているか（`/v2/`）
- [ ] すべての公開エンドポイントにレート制限が設定されているか
- [ ] Firebase JWT の検証がバックエンドで必ず実行されているか（クライアント任せにしない）
- [ ] 廃止予定エンドポイントに `Deprecation` ヘッダーと移行期限を明示しているか
