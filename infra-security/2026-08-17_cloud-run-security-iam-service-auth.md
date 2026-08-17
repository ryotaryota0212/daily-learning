# Cloud Run のセキュリティ設定・IAM・サービス間認証

## 概要

Cloud Run は「コンテナを動かすだけ」に見えるが、セキュリティ設定を誤ると認証なしで全世界に公開・意図しないサービス間アクセスを許可してしまう。  
FastAPI + Firebase Auth + Neon のスタックでは、**フロント → Cloud Run → Neon** の各層で適切なアイデンティティ境界を引くことが可用性とセキュリティの両立に直結する。  
特に「サービスアカウントの権限設計」と「サービス間認証（OIDC トークン）」は、マイクロサービス化した際に最初に詰まるポイント。

---

## 仕組みの要点

### Cloud Run の認証モデル

- **未認証呼び出し許可 (allUsers)**: 誰でも HTTPS でアクセス可。公開 API 向け
- **認証済みのみ (allAuthenticatedUsers)**: Google ID トークンが必要。内部 API 向け
- Cloud Run は IAM で「誰が invoke できるか」を制御する（ネットワーク制御より IAM が優先）

### サービスアカウント設計

- Cloud Run インスタンスには**実行 SA（Runtime SA）**を割り当てる
- Runtime SA の権限 = そのサービスが「できること」。最小権限原則が必須
- デフォルト SA（Compute Engine デフォルト）は使わない → 過剰な権限を持つ

### サービス間認証の仕組み

```
Service A (Cloud Run)
  └─ metadata server から OIDC トークン取得
       aud = https://<service-b-url>
  └─ Authorization: Bearer <token> で Service B を呼ぶ

Service B (Cloud Run)
  └─ IAM で Service A の SA に roles/run.invoker を付与
  └─ Cloud Run が自動でトークン検証 → 通過
```

- トークン取得は GCP メタデータサーバー (`http://metadata.google.internal`) から行う
- `google-auth` ライブラリを使うと 1 関数で取得可能

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| `allUsers` で内部 API を公開 | `allAuthenticatedUsers` + IAM invoker のみ |
| デフォルト SA を使い回す | 用途ごとに専用 SA を作成 |
| SA キーファイル (.json) をコードに含める | Workload Identity / メタデータサーバーを使う |
| 全サービスに `roles/editor` を付与 | 必要な権限だけを個別付与 |
| HTTP で内部通信 | 常に HTTPS（Cloud Run はデフォルト HTTPS） |

---

## コード例（最小限）

### サービス間認証 — FastAPI からの呼び出し側

```python
import google.auth.transport.requests
import google.oauth2.id_token

def call_internal_service(url: str) -> dict:
    auth_req = google.auth.transport.requests.Request()
    token = google.oauth2.id_token.fetch_id_token(auth_req, url)
    import httpx
    resp = httpx.get(url, headers={"Authorization": f"Bearer {token}"})
    resp.raise_for_status()
    return resp.json()
```

### terraform で invoker 権限を付与

```hcl
resource "google_cloud_run_v2_service_iam_member" "invoker" {
  project  = var.project
  location = var.region
  name     = google_cloud_run_v2_service.api_b.name
  role     = "roles/run.invoker"
  member   = "serviceAccount:${google_service_account.api_a_sa.email}"
}
```

---

## トレードオフ

| 選択肢 | メリット | デメリット |
|---|---|---|
| IAM 認証（OIDC） | GCP ネイティブ。管理が IAM に集約 | GCP 外サービスとの連携には不向き |
| API キー | シンプル | ローテーション・漏洩リスクが高い |
| mTLS | 強固な双方向認証 | 証明書管理コストが高い |
| VPC + Private IP | ネットワーク層で隔離できる | Cloud Run の VPC 設定が複雑 |

**推奨**: GCP 内サービス間 → IAM + OIDC。外部パートナーとの連携 → API キー（Secret Manager 管理）

---

## チェックリスト

- [ ] 内部 API の Cloud Run は `allUsers` を外し、`roles/run.invoker` を呼び出し元 SA にのみ付与
- [ ] 各 Cloud Run サービスに専用 Runtime SA を設定（デフォルト SA を使っていない）
- [ ] SA キーファイルを発行していない（Workload Identity / メタデータサーバーを使用）
- [ ] Firebase Auth の ID トークン検証は Cloud Run 上の FastAPI で行い、Neon には直接アクセスさせない
- [ ] Terraform / IaC で IAM 権限を管理し、手動の `gcloud` 操作を本番環境で行わない
