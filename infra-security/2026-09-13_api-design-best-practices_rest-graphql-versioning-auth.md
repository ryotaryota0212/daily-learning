# API設計のベストプラクティス（REST vs GraphQL、バージョニング、レート制限、認証）

## 概要

APIはシステムの「契約」であり、一度公開すると変更コストが高い。AI時代においてもAPIの設計力は変わらず重要で、「とりあえず動く」設計ではなく「壊れにくく・変更しやすく・悪用されにくい」設計を選択できる力が問われる。FastAPI + Cloud Run 構成では特に、認証・バージョニング・レート制限の設計が後から変えにくいため、初期段階での正しい選択が重要。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| ユースケース | CRUD中心・公開API | 複雑なネスト・クライアント主導 |
| オーバーフェッチ | 起きやすい | 起きにくい |
| キャッシュ | HTTP標準で容易 | 複雑（クエリごとに異なる） |
| N+1問題 | 設計で回避可能 | DataLoaderが必要 |
| レート制限 | エンドポイント単位で簡単 | クエリコスト計算が必要 |

**判断基準（FastAPI前提）:**
- 社内API・BFF・モバイル向け → GraphQL を検討
- 公開API・シンプルCRUD・外部連携 → REST を選ぶ
- 迷ったら REST：変更・デバッグ・監視のコストが低い

### バージョニング戦略

- **URLパス方式**（推奨）: `/v1/users`, `/v2/users` — 明示的・キャッシュしやすい
- **ヘッダー方式**: `Accept: application/vnd.api+json;version=2` — URLが汚れないが見えにくい
- **クエリパラメータ方式**: `?version=2` — 避ける（キャッシュ汚染のリスク）

**移行戦略:**
- v1とv2を並走させ、Deprecation-Dateヘッダーで廃止日を通知
- 破壊的変更（フィールド削除・型変更）は必ず新バージョンで行う
- 追加（フィールド追加）は同バージョンで許容

---

## アンチパターン vs 正しい設計

### アンチパターン
- `GET /getUser?id=123` → 動詞をURLに含める
- `POST /api/users` にバージョンなし → 将来の変更が破壊的になる
- 全エンドポイントを認証なしで公開し、後から追加 → 漏洩リスク
- エラーレスポンスが `{"error": "something went wrong"}` だけ → デバッグ不能
- レート制限なしで公開 → DDoS・スクレイピング被害

### 正しい設計
- `GET /v1/users/{user_id}` — RESTful + バージョン付き
- エラーレスポンスはRFC 7807（Problem Details）準拠
- 認証は最初から全エンドポイントに適用し、明示的に外す
- レート制限はアーキテクチャレイヤー（Cloud Run前段）で実装

---

## コード/設計例（FastAPI）

```python
from fastapi import FastAPI, Depends, HTTPException, Request
from fastapi.responses import JSONResponse

app = FastAPI()

# RFC 7807準拠のエラーレスポンス
def problem_detail(status: int, title: str, detail: str):
    return JSONResponse(
        status_code=status,
        content={"type": f"https://api.example.com/errors/{status}",
                 "title": title, "detail": detail, "status": status},
        headers={"Content-Type": "application/problem+json"}
    )

# バージョニング: APIRouterで分離
from fastapi import APIRouter
v1 = APIRouter(prefix="/v1")
v2 = APIRouter(prefix="/v2")

@v1.get("/users/{user_id}")
async def get_user_v1(user_id: str, user=Depends(verify_firebase_token)):
    ...

app.include_router(v1)
app.include_router(v2)
```

---

## トレードオフ

| 設計選択 | メリット | デメリット |
|----------|---------|-----------|
| URLパスバージョニング | 明示的・コピペしやすい | バージョン爆発のリスク |
| GraphQL | 型安全・柔軟 | キャッシュ・レート制限が複雑 |
| Cloud Runでレート制限 | アプリコード外で管理 | 細粒度の制御が難しい |
| 全エンドポイント認証 | 安全 | 公開エンドポイントに手間が増える |

**FastAPI + Cloud Run での推奨構成:**
- レート制限: Cloud Armorまたは上位のCloudflare WAFで実施
- 認証: Firebase Auth JWT検証をDependsで共通化
- バージョニング: APIRouterで `/v1`, `/v2` を分離管理

---

## チェックリスト

- [ ] 全エンドポイントにバージョンプレフィックス（`/v1/`）を付けているか
- [ ] エラーレスポンスはRFC 7807形式（`title`, `detail`, `status`）か
- [ ] 認証（Firebase JWT検証）は`Depends`で全ルートに適用されているか
- [ ] レート制限はアプリ外（Cloud Armor / Cloudflare）で設定されているか
- [ ] 破壊的変更（フィールド削除・型変更）は新バージョンで行っているか
