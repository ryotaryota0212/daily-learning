# API設計のベストプラクティス（REST vs GraphQL・バージョニング・レート制限・認証）

## 概要

APIはシステムの「契約書」であり、一度公開すると変更コストが高い。AI時代においても「どの設計でどのトレードオフを取るか」を説明できる力が求められる。REST/GraphQLの使い分け、安全なバージョニング、レート制限の実装、認証の組み込み方を体系的に理解する。

---

## 仕組みの要点

### REST vs GraphQL 選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いているケース | リソースが明確・キャッシュ重視 | 複雑なデータ取得・モバイル最適化 |
| キャッシュ | HTTP標準でそのまま使える | 追加実装が必要（Persisted Query等） |
| オーバーフェッチ | 発生しやすい | 不要なフィールドを取得しない |
| 学習コスト | 低い | スキーマ設計が必要 |
| 採用基準 | 公開API・シンプルなCRUD | 内部BFF・ダッシュボード系 |

**原則**: 迷ったらREST。GraphQLはBFF（Backend for Frontend）パターンで使うと真価を発揮。

### バージョニング戦略

- **URLパス方式**: `/v1/users` → 最も一般的。明示的で分かりやすい
- **ヘッダー方式**: `Accept: application/vnd.myapp.v1+json` → URLが汚れない
- **廃止ポリシー**: 旧バージョンは最低6ヶ月はサポート継続を明示する

### レート制限設計

- アルゴリズム選択: 固定窓 < スライディング窓 < トークンバケット（推奨）
- 粒度: ユーザーIDベース > IPベース（プロキシ対策）
- レスポンスヘッダー: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `Retry-After`

### 認証の組み込み方（Firebase Auth + Cloud Run）

- 全APIに `Authorization: Bearer <Firebase ID Token>` を要求
- Cloud RunのサービスごとにIAMで最小権限を設定
- サービス間通信は Firebase IDトークンではなくGoogle ID Token（OIDC）を使う

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|-------------|----------|
| `/getUser`, `/createUser`（動詞URL） | `/users` GET/POST（リソース+HTTPメソッド） |
| エラー時も200を返す | 適切なHTTPステータスコード（400/401/404/500） |
| バージョンなし公開API | `/v1/` プレフィックスを初期から付ける |
| レート制限なし | 全エンドポイントにレート制限を実装 |
| トークンをURLに含める | `Authorization` ヘッダーのみ使用 |
| エラーメッセージに内部情報 | 汎用メッセージ＋ログで詳細管理 |

---

## コード/設計例（FastAPI + Firebase Auth）

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer
from firebase_admin import auth
import redis

bearer = HTTPBearer()
r = redis.Redis()

async def verify_token(token=Depends(bearer)):
    try:
        decoded = auth.verify_id_token(token.credentials)
        return decoded
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")

async def rate_limit(user=Depends(verify_token)):
    key = f"rate:{user['uid']}"
    count = r.incr(key)
    if count == 1:
        r.expire(key, 60)  # 1分窓
    if count > 100:
        raise HTTPException(status_code=429, headers={"Retry-After": "60"})
    return user

@app.get("/v1/items")
async def get_items(user=Depends(rate_limit)):
    return {"user_id": user["uid"]}
```

---

## トレードオフ

| 選択 | メリット | デメリット |
|------|---------|----------|
| URLバージョニング | 明示的・デバッグしやすい | URLが冗長 |
| トークンバケット | バースト許容・スムーズ | Redis依存 |
| GraphQL | 柔軟なクエリ | N+1問題・Complexity攻撃リスク |
| JWT（ステートレス） | スケールしやすい | トークン失効が困難 |

---

## チェックリスト

- [ ] URLに動詞を使わず、HTTPメソッドでCRUDを表現している
- [ ] `/v1/` プレフィックスを初期から付けている
- [ ] レート制限をユーザーIDベースで実装している（429 + Retry-After）
- [ ] 認証はAuthorizationヘッダーのみ。URLにトークンを含めない
- [ ] エラーレスポンスに内部実装の情報（スタックトレース等）を含めない
