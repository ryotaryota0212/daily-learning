# API設計のベストプラクティス（REST vs GraphQL・バージョニング・レート制限・認証）

## 概要

AI時代においてAPIは「システム間の契約」だ。設計が甘いAPIは後から直せず、クライアントを壊し、セキュリティホールになる。
FastAPI + Cloud Run 構成では「誰に何を公開するか」「どう壊れても安全か」を最初に設計する必要がある。
`とりあえずエンドポイントを生やす`ではなく、`変更に強く・壊れにくい契約`を設計する力が問われる。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いているケース | CRUD中心、単純なリソース操作 | 複数リソースを組み合わせて取得 |
| クライアントの柔軟性 | 低（サーバーが形を決める） | 高（クライアントが形を決める） |
| キャッシュのしやすさ | 簡単（URL単位） | 難しい（POST固定） |
| 学習コスト | 低 | 高 |
| 過剰取得・不足取得 | 起きやすい | 起きにくい |

**判断基準：** モバイル・BFF（Backend for Frontend）がある → GraphQL。社内API・シンプルなCRUD → REST。

### バージョニング戦略

- **URLパス方式**（`/v1/users`）：最もシンプルで推奨。可視性が高い
- **ヘッダー方式**（`Accept: application/vnd.api+v2`）：URLを汚さないが複雑
- **クエリパラメータ方式**（`?version=2`）：非推奨。キャッシュが効きにくい

**原則：** 後方互換を破る変更 = メジャーバージョンアップ。フィールドの追加は後方互換OK。

### 認証設計

- Firebase Auth のIDトークン（JWT）を `Authorization: Bearer <token>` で送る
- Cloud RunのサービスアカウントでサービスID認証（サービス間通信）
- エンドポイントごとに「認証必須 / オプション / 不要」を明示的に設計する

---

## アンチパターン vs 正しい設計

### アンチパターン
- `GET /getUsers`、`POST /deleteUser` → 動詞をURLに入れる（HTTPメソッドの役割を無視）
- `200 OK` でエラーを返す → クライアントがステータスコードを信頼できない
- 認証チェックをルーターではなく各ハンドラに書く → 漏れが生まれる
- フィールド名に型情報を埋める → `userIdInt`, `createdAtString`

### 正しい設計
- リソース名は複数形の名詞：`GET /users`, `POST /users`, `DELETE /users/{id}`
- ステータスコードを正しく使う：`201 Created`, `404 Not Found`, `422 Unprocessable Entity`
- 認証は依存注入でミドルウェア化する
- エラーレスポンスは統一フォーマット

---

## コード例（FastAPI + Firebase Auth）

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import firebase_admin.auth as auth

bearer = HTTPBearer()

def verify_token(creds: HTTPAuthorizationCredentials = Depends(bearer)):
    try:
        return auth.verify_id_token(creds.credentials)
    except Exception:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED)

# エラーレスポンスの統一フォーマット
class ErrorResponse(BaseModel):
    code: str
    message: str

@app.get("/v1/users/{user_id}", responses={404: {"model": ErrorResponse}})
async def get_user(user_id: str, claims=Depends(verify_token)):
    ...
```

---

## レート制限設計

- **目的別に閾値を分ける**：認証エンドポイント（厳しく）、読み取りAPI（緩め）
- **ユーザー単位 / IP単位 / グローバル** の3層で設計
- Cloud Run の前段に **Cloudflare Rate Limiting** を置くのが最もシンプル
- レート制限超過は `429 Too Many Requests` + `Retry-After` ヘッダーで返す

---

## トレードオフ

| 選択 | メリット | デメリット |
|------|---------|---------|
| URLバージョニング | 明示的・キャッシュしやすい | URLが長くなる |
| GraphQL | 過剰取得なし | N+1問題・認証が複雑 |
| JWT認証 | ステートレス | トークン失効が即時できない |
| 厳格なレート制限 | 悪用防止 | 正当なバーストに影響 |

---

## チェックリスト

- [ ] エンドポイントはリソース名（名詞・複数形）で命名されているか
- [ ] 認証チェックはミドルウェアで一元管理されているか
- [ ] エラーレスポンスは全エンドポイントで統一フォーマットか
- [ ] レート制限はエンドポイントの役割別に設定されているか
- [ ] バージョン変更ポリシー（後方互換の定義）がドキュメント化されているか
