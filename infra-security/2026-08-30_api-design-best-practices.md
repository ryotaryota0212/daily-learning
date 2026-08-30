# API設計のベストプラクティス（REST vs GraphQL、バージョニング、レート制限、認証）

## 概要

APIはシステムの「契約」であり、一度公開すると変更コストが高い。設計の誤りはクライアントへの破壊的変更、セキュリティホール、スケーラビリティ問題に直結する。AI時代においても「どんな契約を結ぶか設計する力」は人間が担う中核スキルであり、「とりあえず動くエンドポイント」ではなく「変化に強く壊れにくいAPI」を設計できることが重要。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|---|---|---|
| 向いているケース | CRUD中心・公開API・キャッシュ重視 | 複雑なデータ取得・BFF・モバイル |
| Over-fetching | 起きやすい | 起きにくい |
| キャッシュ | HTTP標準キャッシュが使える | クエリ単位で複雑 |
| スキーマ | OpenAPIで別途定義 | 型システムが組み込み |
| 運用コスト | 低い | 高い（resolver N+1問題等） |

**FastAPI + Neon スタックでは原則REST**。GraphQLはBFF層が必要な複雑なSPAや、複数リソースを柔軟に組み合わせる要件が明確な場合のみ採用を検討。

### バージョニング戦略

- **URLパス方式**（`/v1/users`）: 最もわかりやすく、ルーティングが明確。推奨
- **ヘッダー方式**（`API-Version: 2024-01-01`）: URLを汚さないがテストしにくい
- **クエリパラメータ方式**（`?version=2`）: ブックマーク可だがキャッシュが複雑に

**バージョンを上げる基準**: レスポンスの削除・型変更・必須パラメータの追加。追加のみなら後方互換で同一バージョン継続可。

### レート制限の設計

- **固定ウィンドウ**: シンプルだが境界バーストに弱い
- **スライディングウィンドウ**: バースト防止に有効、Redis実装が標準的
- **トークンバケット**: APIゲートウェイで多く採用。リクエストのバースト許容も可能

レート制限のレイヤー:
1. Cloudflare（WAFレベル）: DDoS・Bot対策
2. Cloud Run前段（Cloud Endpoints / Load Balancer）: IPベース
3. アプリ層（FastAPI middleware）: ユーザー・APIキー単位

### 認証パターン

- Firebase Auth の `id_token` をBearerで渡す → FastAPIのDependencyで検証
- サービス間通信はService Account + IDトークン（Cloud Run → Cloud Run）
- 外部向けはAPIキー + レート制限（Stripeモデルが参考になる）

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| `GET /getUserById?id=1` | `GET /users/{id}` |
| ステータスは常に200で本文にerrorフラグ | HTTPステータスコードを正しく使う |
| バージョニングなしで破壊的変更 | `/v1/` をURLに含め、旧バージョンをdeprecation期間付きで維持 |
| エラーレスポンスが統一されていない | RFC 9457 (Problem Details) 形式で統一 |
| 認証なしの管理系エンドポイント | すべてのエンドポイントに認証・認可チェック |
| N+1クエリをそのままAPIに | JOIN or DataLoaderでバッチ取得 |

---

## コード/設計例（FastAPI）

```python
# レート制限 + Firebase認証 + バージョニングの最小構成
from fastapi import FastAPI, Depends, HTTPException, Request
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
app = FastAPI()

async def verify_token(request: Request) -> dict:
    token = request.headers.get("Authorization", "").removeprefix("Bearer ")
    if not token:
        raise HTTPException(status_code=401)
    # firebase_admin.auth.verify_id_token(token) で検証
    return {"uid": "user_id"}

@app.get("/v1/items/{item_id}")
@limiter.limit("60/minute")
async def get_item(item_id: int, request: Request, user=Depends(verify_token)):
    return {"id": item_id, "owner": user["uid"]}

# エラーレスポンス統一（RFC 9457 Problem Details）
def problem(status: int, title: str, detail: str):
    return {"type": f"/errors/{title}", "status": status, "detail": detail}
```

---

## トレードオフ

- **RESTのリソース単位 vs GraphQLの柔軟性**: クライアント要件が画一的ならREST、UI都合が複雑なら検討。ただしGraphQLの運用コスト（N+1、キャッシュ、認可の複雑さ）を過小評価しない
- **バージョニング方針**: URLパスは管理しやすいが、N個のバージョンを並行運用するとコードが膨らむ。廃止スケジュールを最初から決めておく
- **レート制限の粒度**: IPベースは共有IPで誤爆しやすい。ユーザー単位が理想だが認証前のエンドポイント（/auth等）はIP併用が必要
- **後方互換性 vs 技術的負債**: 古いバージョンを長く維持するほど負債が増える。Deprecation通知→6ヶ月後削除のサイクルを標準化する

---

## チェックリスト

- [ ] エンドポイントURLはリソース名詞・HTTPメソッドで動詞を表現している
- [ ] 全エンドポイントにFirebase Auth検証Dependencyが適用されている
- [ ] レート制限がCloudflare / アプリ層の2段階で設定されている
- [ ] エラーレスポンスがRFC 9457形式に統一されている
- [ ] 破壊的変更時のURLバージョニング（`/v1/`, `/v2/`）とdeprecation期間が定義されている
