# Cloud Run のセキュリティ設定・IAM・サービス間認証

## 概要

Cloud Run はサーバーレスコンテナ実行環境だが、**デフォルト設定は「全公開」に近い**。
適切な IAM 設計とサービス間認証を施さないと、意図しない外部アクセスや権限昇格を招く。
FastAPI + Cloud Run 構成では「誰がこのサービスを呼べるか」を IAM で明示的に制御することが設計の基本となる。

---

## 仕組みの要点

### IAM の基本構造

- **サービスアカウント**: Cloud Run サービスが「誰として動くか」を決める ID
- **IAM バインディング**: `誰が（member）` `何を（role）` `どこに（resource）` できるかを定義
- **Caller 認証**: 呼び出し元が正規のサービスかを `Authorization: Bearer <ID Token>` で検証

### Cloud Run 固有の設定項目

| 設定 | 安全な値 | 危険な値 |
|------|----------|----------|
| 未認証アクセス | 拒否（require auth） | 許可（allow unauthenticated） |
| サービスアカウント | 専用 SA（最小権限） | Default Compute SA |
| VPC コネクタ | 内部通信は VPC 経由 | 全て公開エンドポイント |
| イングレス | `internal-and-cloud-load-balancing` | `all` |

---

## アンチパターン vs 正しい設計

### アンチパターン

- `allUsers` に `roles/run.invoker` を付与 → 誰でも呼び出せる
- Default Compute Service Account を使う → 過剰な権限（GCS, BigQuery等に全アクセス）
- サービス間通信でAPIキーを使う → 漏洩リスク、ローテーション困難
- 全サービスを同一サービスアカウントで動かす → 侵害時に全サービスが危険に

### 正しい設計

- サービスごとに専用 SA を作成し、必要最小限のロールのみ付与
- サービス間通信は **OIDC ID Token** を使った認証に統一
- 外部公開は Cloud Load Balancer + Cloud Armor 経由のみ
- イングレスを `internal` に絞り、直接呼び出しを遮断

---

## コード/設計例

### サービス間認証（FastAPI 呼び出し側）

```python
import google.auth.transport.requests
import google.oauth2.id_token
import httpx

async def call_internal_service(url: str) -> dict:
    auth_req = google.auth.transport.requests.Request()
    token = google.oauth2.id_token.fetch_id_token(auth_req, url)
    async with httpx.AsyncClient() as client:
        resp = await client.get(
            url,
            headers={"Authorization": f"Bearer {token}"}
        )
    resp.raise_for_status()
    return resp.json()
```

### 受け取り側の検証（FastAPI）

```python
from fastapi import Depends, HTTPException, Header
from google.oauth2 import id_token
from google.auth.transport import requests as grequests

def verify_id_token(authorization: str = Header(...)):
    token = authorization.removeprefix("Bearer ")
    try:
        info = id_token.verify_oauth2_token(
            token, grequests.Request()
        )
        if info["email"] not in ALLOWED_SERVICE_ACCOUNTS:
            raise HTTPException(status_code=403)
        return info
    except Exception:
        raise HTTPException(status_code=401)
```

### Terraform: 最小権限 SA 設定例

```hcl
resource "google_service_account" "api_sa" {
  account_id = "fastapi-service"
}

resource "google_cloud_run_service_iam_member" "invoker" {
  service  = google_cloud_run_v2_service.backend.name
  role     = "roles/run.invoker"
  member   = "serviceAccount:${google_service_account.api_sa.email}"
}
```

---

## トレードオフ

| 選択肢 | メリット | デメリット |
|--------|----------|------------|
| ID Token 認証 | Google マネージド、自動ローテーション | 実装コストあり、ライブラリ依存 |
| API Key | シンプル | 漏洩リスク、手動ローテーション必須 |
| VPC 内部通信のみ | ネットワーク層で遮断 | VPC コネクタのコスト発生 |
| allUsers 公開 | 実装が楽 | 認証なしで誰でも呼べる（本番NG） |

**FastAPI + Cloud Run の推奨**: サービス間は ID Token、外部公開は Firebase Auth の JWT 検証を組み合わせる

---

## チェックリスト

- [ ] Cloud Run サービスごとに専用サービスアカウントを作成している
- [ ] `allUsers` への `roles/run.invoker` 付与がないことを確認している
- [ ] サービス間通信は OIDC ID Token で認証している
- [ ] イングレスを `internal-and-cloud-load-balancing` 以下に制限している
- [ ] Default Compute Service Account を Cloud Run に使用していない
