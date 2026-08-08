# API設計のベストプラクティス — REST vs GraphQL・バージョニング・レート制限・認証

## 概要

AIがコードを書く時代でも「どういうAPIを設計するか」の判断は人間が担う。
エンドポイント設計の失敗はクライアントとの契約違反になり、後から直すコストが膨大になる。
スケール・セキュリティ・保守性を兼ねた設計が、長期的なシステムの健全性を左右する。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 適合ケース | リソース単位でCRUDが明確 | フロントが取得フィールドを動的に選ぶ |
| N+1問題 | エンドポイント設計で回避 | DataLoader必須 |
| キャッシュ | HTTP標準で容易 | POSTベースで難しい |
| 型安全 | OpenAPIで補完 | スキーマが契約になる |

**FastAPI + Neon スタックでは REST を基本とし、BFF層が複数リソースを集約する場合のみ GraphQL を検討する。**

### バージョニング戦略

- **URLパス方式**（`/v1/users`）— 最も明示的。破壊的変更時に新バージョンを切る
- **ヘッダー方式**（`Accept: application/vnd.api+json;version=2`）— URL汚染なし、テストしにくい
- **クエリパラメータ方式**（`?version=2`）— 避ける（キャッシュの挙動が複雑になる）

**推奨: URLパス方式 + 旧バージョンは6ヶ月間 Deprecation Header を返す**

### レート制限の設計

- ユーザー単位 / IPアドレス単位 / テナント単位を分離して制限
- Cloud Run の場合、上流に Cloudflare または API Gateway を置いてエッジで遮断
- Redis + トークンバケット or スライディングウィンドウで実装

### 認証の組み込み方

- Firebase Auth の JWT を `Authorization: Bearer <token>` で受け取る
- FastAPI の `Depends()` でデコード・検証を一元化
- サービス間通信は Cloud Run の OIDC トークンを使いサービスアカウントで認証

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|----------------|-----------|
| `GET /getUser?id=1` | `GET /users/{id}` |
| エラーを全部 `200 OK` で返す | `400/401/403/404/500` を使い分ける |
| レスポンスに内部DB構造をそのまま返す | DTOで公開フィールドを明示 |
| バージョニングなしで破壊的変更 | `/v2/` に切り出しクライアントに移行期間を与える |
| 認証チェックをコントローラーに直書き | `Depends()` ミドルウェアで強制 |

---

## コード例（FastAPI + Firebase Auth）

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from firebase_admin import auth

bearer = HTTPBearer()

def get_current_user(creds: HTTPAuthorizationCredentials = Depends(bearer)):
    try:
        decoded = auth.verify_id_token(creds.credentials)
        return decoded
    except Exception:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED)

@app.get("/v1/users/{user_id}")
def get_user(user_id: str, user=Depends(get_current_user)):
    if user["uid"] != user_id:
        raise HTTPException(status_code=403)
    return fetch_user(user_id)
```

---

## トレードオフ

| 選択 | メリット | デメリット |
|------|---------|-----------|
| RESTのURLバージョニング | 明確・移行しやすい | URL設計が冗長になる |
| GraphQL | フロント自由度高い | キャッシュ・N+1・認可制御が複雑 |
| エッジでのレート制限 | バックエンド到達前に遮断 | ルール変更にデプロイ不要だが設定が分散 |
| JWT検証を毎リクエスト | ステートレスでスケールしやすい | 鍵更新のラグがある（`jwks_uri` でローテーション対応） |

---

## チェックリスト

- [ ] HTTPメソッドとステータスコードをREST規約に沿って使っている
- [ ] `/v1/` でバージョニングし、破壊的変更時は `/v2/` に切り出す方針を決めている
- [ ] 認証は `Depends()` で全エンドポイントに強制されており、バイパスできない
- [ ] レート制限をユーザー単位で設定し、429 Too Many Requests を返している
- [ ] レスポンスに内部フィールド（DBのIDや実装詳細）を露出していない
