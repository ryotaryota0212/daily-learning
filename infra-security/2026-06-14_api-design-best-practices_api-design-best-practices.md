# API設計のベストプラクティス — REST vs GraphQL・バージョニング・レート制限・認証

## 概要

API設計はシステムの「契約」であり、一度公開すると変更コストが非常に高い。  
AI時代でも「どういうAPIにするか決める力」はエンジニアに求められ続ける。  
REST・GraphQLの使い分け、安全なバージョニング、適切なレート制限・認証設計を理解することで、スケールしても壊れない境界線を引けるようになる。

---

## 仕組みの要点

### REST vs GraphQL の選択軸

| 観点 | REST | GraphQL |
|---|---|---|
| クライアント多様性 | 低い（Web/Mobile同一） | 高い（必要フィールドが異なる） |
| over-fetching問題 | 起きやすい | 起きない |
| キャッシュ | HTTP標準キャッシュが効く | クエリ単位でカスタム必要 |
| 学習コスト | 低い | スキーマ設計の知識が必要 |
| ファイルアップロード | 自然 | 複雑（multipart拡張が必要） |

**判断基準**
- クライアントが1種類でデータ形状が固定 → REST
- モバイル/Web/サードパーティが同一APIを使い、フィールドが異なる → GraphQL
- 公開APIでドキュメント重視 → REST（OpenAPI連携しやすい）

### バージョニング戦略

- **URLパス方式** `/api/v1/users`：最も一般的。可視性高い
- **ヘッダー方式** `Accept: application/vnd.api.v2+json`：URLをきれいに保てる
- **非推奨化フロー**：`Deprecation` / `Sunset` レスポンスヘッダーで廃止予告 → 6ヶ月猶予 → 削除

### レート制限の設計

- **対象単位**：IPアドレス単位（未認証）、ユーザーID単位（認証済み）
- **アルゴリズム**：Token Bucket（バースト許容）vs Fixed Window（実装シンプル）
- **レスポンス**：HTTP 429 + `Retry-After` / `X-RateLimit-*` ヘッダー
- **Cloud Run + Firebase Auth 構成**：Cloud Armor or Cloudflare でIP制限、アプリ層でユーザー別制限

### 認証設計

- **Firebase Auth JWT を使う場合**：Authorization: Bearer <token> → バックエンドで公開鍵検証
- **サービス間認証（Cloud Run）**：OIDC トークンを使い IAM で制御（APIキーは使わない）
- **スコープ設計**：最小権限原則、読み取り専用スコープと書き込みスコープを分離

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| `/api/getUser`, `/api/deleteUser`（動詞URL） | `GET /users/{id}`, `DELETE /users/{id}` |
| v1/v2 を永遠に両方維持 | Sunset ヘッダーで廃止スケジュールを明示 |
| 全エンドポイントに同一レート制限 | 書き込み系は厳しく、読み取り系は緩く設定 |
| JWT をクライアントで検証せず信頼 | サーバー側で署名検証 + expiry チェック必須 |
| エラーに内部スタックトレースを返す | 外部向けには汎用エラーコード、内部にのみ詳細ログ |

---

## コード例（FastAPI + Firebase Auth）

```python
from fastapi import Depends, HTTPException, Request
from firebase_admin import auth
import time

# Firebase JWT 検証ミドルウェア
async def verify_token(request: Request):
    token = request.headers.get("Authorization", "").removeprefix("Bearer ")
    try:
        decoded = auth.verify_id_token(token)
        return decoded
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")

# レート制限（簡易インメモリ）
_rate_store: dict = {}
def rate_limit(uid: str, limit: int = 60, window: int = 60):
    now = int(time.time())
    bucket = _rate_store.setdefault(uid, {"count": 0, "reset": now + window})
    if now > bucket["reset"]:
        bucket.update({"count": 0, "reset": now + window})
    bucket["count"] += 1
    if bucket["count"] > limit:
        raise HTTPException(status_code=429, headers={"Retry-After": str(bucket["reset"] - now)})

@app.get("/api/v1/users/{user_id}")
async def get_user(user_id: str, claims=Depends(verify_token)):
    rate_limit(claims["uid"])
    # ...
```

---

## トレードオフ

- **バージョニング「URLパス vs ヘッダー」**：URLパスはキャッシュとログが楽。ヘッダーはURL設計がシンプルになるが、クライアント実装が複雑化
- **GraphQLの柔軟性 vs 複雑性**：クエリの自由度が高い分、N+1問題やコスト計算（query depth limit）が必要になる
- **レート制限「アプリ層 vs インフラ層」**：Cloud Armorはユーザー粒度制限が難しい、アプリ層は全サービス共通化が難しい。両方を組み合わせるのがベスト

---

## チェックリスト

- [ ] HTTPメソッドとステータスコードをRFCに従って使えているか（PATCH/PUT の違い含む）
- [ ] バージョン廃止の際に `Sunset` ヘッダーで事前通知しているか
- [ ] 書き込みエンドポイントと読み取りエンドポイントでレート制限を分けているか
- [ ] エラーレスポンスに内部情報（スタックトレース・DB構造）が漏れていないか
- [ ] Firebase Auth JWTの検証（署名 + exp + aud）をサーバー側で必ず実施しているか
