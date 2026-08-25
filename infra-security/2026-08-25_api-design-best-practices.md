# API設計のベストプラクティス（REST vs GraphQL・バージョニング・レート制限・認証）

## 概要

AI時代でもAPIは全システムの「契約」であり、一度公開すると変更コストが極大になる。  
REST/GraphQLの選択・バージョニング戦略・認証方式を最初に正しく決めることが、後の技術的負債を防ぐ最重要の設計判断。  
「とりあえず動くエンドポイント」から「壊れにくく進化できるAPI」への思考転換が目標。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いているケース | 公開API、シンプルなCRUD、キャッシュ重視 | 複雑な関連データ取得、BFF、モバイル最適化 |
| N+1問題 | 複数エンドポイント呼び出しで発生しやすい | DataLoaderで解決可能 |
| クライアント制御 | サーバー側でフィールド固定 | クライアントが必要フィールドを指定 |
| キャッシュ | HTTP標準キャッシュが使える | クエリ単位でキャッシュが複雑 |
| 学習コスト | 低い | スキーマ・リゾルバー設計が必要 |

**FastAPI + Neon スタックでの推奨**：外部公開APIはREST、内部BFF層はGraphQL（Strawberry）を検討。

### バージョニング戦略

- **URLパス方式**（`/v1/users`）：最も一般的・可視性高い。推奨。
- **ヘッダー方式**（`API-Version: 2026-01`）：URLを汚さないがルーティングが複雑。
- **クエリパラメータ方式**（`?version=2`）：デバッグしやすいがキャッシュ汚染リスク。

破壊的変更の判断基準：
- フィールド削除・型変更 → 新バージョン必須
- フィールド追加・デフォルト値変更 → 後方互換なら同バージョン可
- 古いバージョンのサポート期限は**最低6か月**を明示する

### 認証方式の設計

```
クライアント ─→ Firebase Auth でJWT取得
   │
   ↓
Cloud Run (FastAPI) ─→ JWTをFirebase Admin SDKで検証
   │                    ├── uidをコンテキストに乗せる
   ↓                    └── Neon RLS で uid フィルタ
Neon PostgreSQL (RLS)
```

- サービス間通信は`Authorization: Bearer <token>`＋Cloud Run サービスアカウント
- 公開エンドポイントはレート制限必須（未認証でも叩ける）

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---------------|-----------|
| `/getUser`・`/createUser` のような動詞URI | `GET /users/{id}`・`POST /users`（名詞＋HTTPメソッド） |
| 200 OKでエラーを返す | 4xx/5xxを適切に使いProblem Details（RFC 9457）形式で返す |
| バージョンなしで破壊的変更をデプロイ | `/v1/` → `/v2/` に切り替え、並行稼働期間を設ける |
| 認証なしエンドポイントにレート制限なし | Cloudflare WAF + アプリ層の二重レート制限 |
| エラーにスタックトレースを含める | 本番環境では`detail`は一般メッセージのみ、ログにだけ詳細を出す |

---

## コード/設計例（FastAPI）

```python
from fastapi import FastAPI, Depends, HTTPException, Request
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
app = FastAPI()

# Problem Details 形式のエラーレスポンス
def problem(status: int, title: str, detail: str):
    raise HTTPException(status_code=status,
        headers={"Content-Type": "application/problem+json"},
        detail={"type": f"https://example.com/errors/{title}",
                "title": title, "status": status, "detail": detail})

# レート制限付きエンドポイント（v1プレフィックス）
@app.get("/v1/users/{user_id}")
@limiter.limit("100/minute")
async def get_user(request: Request, user_id: str,
                   uid: str = Depends(verify_firebase_token)):
    if uid != user_id:
        problem(403, "forbidden", "Access denied")
    # ...
```

---

## トレードオフ

- **RESTの過剰なエンドポイント増加** vs **GraphQLの複雑なスキーマ管理**：チームのGraphQL習熟度で判断
- **厳格なバージョニング** vs **後方互換維持のコスト**：クライアントが多いほどバージョン管理が安全
- **細かいレート制限** vs **正常ユーザーへの影響**：ユーザー種別（認証済み/匿名）で閾値を分ける
- **詳細なエラーメッセージ** vs **情報漏洩リスク**：本番は汎用メッセージ、開発環境のみ詳細

---

## チェックリスト

- [ ] URIは名詞・階層構造・複数形で統一（`/v1/users/{id}/posts`）
- [ ] HTTPステータスコードを正しく使い、Problem Details形式でエラーを返す
- [ ] バージョンプレフィックス（`/v1/`）を最初から付ける
- [ ] 認証済み/未認証で異なるレート制限を設定する
- [ ] 破壊的変更の定義とサポート期限ポリシーをREADMEに記載する
