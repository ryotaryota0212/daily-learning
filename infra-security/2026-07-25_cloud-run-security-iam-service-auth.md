# Cloud Run のセキュリティ設定・IAM・サービス間認証

## 概要

Cloud Run はサーバーレスコンテナ基盤だが、「デプロイできた＝安全」ではない。
デフォルト設定のままでは誰でもアクセスできる状態になる。
IAM・サービスアカウント・認証トークンの設計を正しく行わないと、
内部APIが外部から叩かれたり、サービス間で過剰な権限が付与される。

---

## 仕組みの要点

### 公開制御（Invoker 権限）
- `roles/run.invoker` を **allUsers** に付与 → 誰でも叩けるパブリックエンドポイント
- `roles/run.invoker` を **特定のサービスアカウント** のみに付与 → 認証必須
- Firebase Auth ユーザー向けの公開APIは `allUsers` でOK。内部APIは絶対NG

### サービス間認証（Service-to-Service）
1. 呼び出し元サービスがIDトークンを取得（対象URLのaudienceを指定）
2. リクエストヘッダーに `Authorization: Bearer <IDトークン>` を付与
3. Cloud Run がトークンを検証 → 正当なサービスアカウントからのみ許可

### サービスアカウント設計（最小権限原則）
- 各Cloud Runサービスに専用のサービスアカウントを割り当てる
- デフォルトサービスアカウントは使わない（過剰権限）
- Neon（PostgreSQL）接続はDB接続文字列で管理、IAM直接接続は不要

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| デフォルトSAをそのまま使用 | 専用SA + 最小権限のみ付与 |
| 内部APIに `allUsers` を設定 | 内部APIはInvoker権限を限定 |
| サービス間でAPIキーを使う | IDトークン（OIDC）で認証 |
| シークレットを環境変数にベタ書き | Secret Manager を参照 |
| `--allow-unauthenticated` を全サービスに | 公開APIのみに限定 |

---

## コード例

### FastAPI側：受信トークンの検証（内部API）

```python
from fastapi import HTTPException, Request
import google.auth.transport.requests
import google.oauth2.id_token

async def verify_service_token(request: Request):
    auth = request.headers.get("Authorization", "")
    if not auth.startswith("Bearer "):
        raise HTTPException(status_code=401)
    token = auth.split(" ")[1]
    try:
        info = google.oauth2.id_token.verify_oauth2_token(
            token,
            google.auth.transport.requests.Request(),
            audience="https://my-internal-service-xxxx-an.a.run.app"
        )
    except Exception:
        raise HTTPException(status_code=403)
    return info
```

### 呼び出し元：IDトークン取得＆リクエスト

```python
import google.auth.transport.requests
import google.oauth2.id_token
import requests as http_requests

TARGET_URL = "https://my-internal-service-xxxx-an.a.run.app"

def call_internal_api():
    auth_req = google.auth.transport.requests.Request()
    token = google.oauth2.id_token.fetch_id_token(auth_req, TARGET_URL)
    resp = http_requests.get(
        f"{TARGET_URL}/internal/data",
        headers={"Authorization": f"Bearer {token}"}
    )
    return resp.json()
```

---

## トレードオフ

| 選択肢 | メリット | デメリット |
|---|---|---|
| allUsers（公開） | 設定不要、Firebase Authでアプリ側制御 | Cloud Run層での防御がない |
| IAM認証（内部） | インフラ層で確実にブロック | 呼び出し元の実装コストがかかる |
| API Gateway経由 | 統一的なレート制限・認証 | コストと管理コスト増 |
| VPC内部通信 | ネットワーク層で隔離 | Cloud Run VPCコネクタ設定が必要 |

**推奨構成（FastAPI + Cloud Run + Firebase Auth）:**
- フロントエンド向けAPI → `allUsers` + Firebase Auth トークン検証（アプリ層）
- サービス間API（バックグラウンドジョブ、内部処理）→ IAM Invoker + IDトークン

---

## チェックリスト

- [ ] 内部APIに `allUsers` が付いていないか確認
- [ ] 各サービスに専用サービスアカウントを割り当てている
- [ ] シークレット（DB URL等）はSecret Managerから参照している
- [ ] サービス間呼び出しはIDトークン（OIDC）で認証している
- [ ] デプロイ時に `--no-allow-unauthenticated` がデフォルトになっているか確認
