# Cloud Run セキュリティ設定・IAM・サービス間認証

## 概要

Cloud Run はサーバーレスでスケールするが、デフォルト設定のまま使うと認証なしの公開エンドポイントになる。
FastAPI + Firebase Auth + Neon のスタックでは、①フロントエンドからの認証済みリクエスト処理、②サービス間の内部通信、③シークレット管理の3つが設計の要。
IAM を正しく設定しないと「認証バイパス」「権限昇格」「内部 API の意図しない公開」が起きる。

---

## 仕組みの要点

### Cloud Run の認証モデル

- **未認証アクセス許可あり**: 誰でも叩ける（BFF やパブリック API 向け）
- **未認証アクセス許可なし（推奨）**: IAM `roles/run.invoker` を持つプリンシパルのみ呼び出せる
- サービスアカウントに invoker ロールを付与 → サービス間通信は OIDC トークンで認証

### Firebase Auth との連携

- フロントエンドは Firebase Auth で ID トークン取得 → `Authorization: Bearer <idToken>` で送信
- Cloud Run 上の FastAPI が `firebase_admin` でトークン検証 → UID / claims を取り出す
- **Cloud Run 自体の認証と Firebase ID トークン検証は別レイヤー**（混同注意）

### サービス間認証（サービスA → サービスB）

1. サービスAのサービスアカウントにサービスBの `roles/run.invoker` を付与
2. サービスA が Google メタデータサーバーから OIDC トークンを取得
3. トークンを `Authorization: Bearer` に付けてサービスBを呼ぶ
4. Cloud Run が自動でトークン検証（コード不要）

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| 全サービスに未認証アクセス許可 | パブリックエンドポイントのみ許可、内部は IAM 必須 |
| デフォルトのCompute Service Account を使う | サービスごとに専用 SA を作成し最小権限 |
| 環境変数でシークレットを直接埋め込む | Secret Manager でマウント、SA に secretAccessor を付与 |
| サービス間で API キーを共有 | OIDC トークンで都度認証（有効期限 1h） |
| 全トラフィックをインターネット経由 | VPC Connector + Private IP でサービス間を内部通信 |

---

## コード例（最小限）

### FastAPI で Firebase ID トークンを検証

```python
import firebase_admin
from firebase_admin import auth, credentials
from fastapi import HTTPException, Header

cred = credentials.ApplicationDefault()
firebase_admin.initialize_app(cred)

async def verify_token(authorization: str = Header(...)):
    if not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401)
    token = authorization.removeprefix("Bearer ")
    try:
        decoded = auth.verify_id_token(token)
        return decoded  # uid, email, custom_claims
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")
```

### サービス間通信（OIDC トークン取得）

```python
import google.auth.transport.requests
import google.oauth2.id_token

def get_id_token(target_url: str) -> str:
    auth_req = google.auth.transport.requests.Request()
    return google.oauth2.id_token.fetch_id_token(auth_req, target_url)

# 呼び出し側
token = get_id_token("https://service-b-xxxxx-an.a.run.app")
headers = {"Authorization": f"Bearer {token}"}
```

### Terraform で IAM 設定（最小権限）

```hcl
resource "google_service_account" "api_sa" {
  account_id = "fastapi-service"
}

resource "google_cloud_run_service_iam_member" "invoker" {
  service  = google_cloud_run_service.service_b.name
  role     = "roles/run.invoker"
  member   = "serviceAccount:${google_service_account.api_sa.email}"
}
```

---

## トレードオフ

| 観点 | 選択肢A | 選択肢B |
|---|---|---|
| サービス間認証 | OIDC（IAM） | API キー共有 |
| メリット | 鍵管理不要・自動ローテーション | 実装が単純 |
| デメリット | コード量が若干増える | 漏洩リスク・ローテーション困難 |
| **推奨** | **本番は OIDC 一択** | 開発ローカルのみ |

**Secret Manager vs 環境変数**: Secret Manager は変更時に再デプロイ不要・アクセスログが残る。環境変数はシンプルだがイメージに焼き込むリスクあり。シークレット類（DB パスワード、API キー）は常に Secret Manager を使う。

---

## チェックリスト

- [ ] 内部サービスはすべて「未認証アクセス不可」に設定されているか
- [ ] サービスごとに専用のサービスアカウントを作り、最小権限になっているか
- [ ] シークレット（DB 接続文字列等）は Secret Manager からマウントしているか
- [ ] サービス間通信は OIDC トークン経由になっているか
- [ ] Cloud Audit Logs（Admin Activity / Data Access）が有効になっているか
