# Cloud Run のセキュリティ設定・IAM・サービス間認証

## 概要

Cloud Run はサーバーレスコンテナ実行環境だが、「デプロイすれば動く」と油断するとセキュリティホールになる。
IAM の最小権限設定・サービス間の認証・シークレット管理を正しく設計しないと、
内部 API が外部から呼び放題、権限昇格、シークレット漏洩などのリスクが生じる。
FastAPI + Cloud Run 構成で押さえるべき設計パターンをまとめる。

---

## 仕組みの要点

### Cloud Run の認証モデル

- **unauthenticated アクセス**: デフォルトは `--allow-unauthenticated` で全公開。フロント向け以外は禁止
- **IAM Invoker**: `roles/run.invoker` を持つサービスアカウントだけが呼び出せる
- **Identity Token**: サービス間通信は `Authorization: Bearer <id_token>` を使う（GCP が発行）
- **Firebase Auth トークンとの違い**: Firebase JWT はユーザー認証用。サービス間通信には OIDC ID Token を使う

### サービスアカウントの設計

- Cloud Run サービスごとに専用のサービスアカウントを作る（デフォルト SA は使わない）
- 付与するロールは最小限。Neon への接続なら Secret Accessor のみ
- Workload Identity は GKE 用。Cloud Run では SA をランタイムにアタッチする形

### シークレット管理

- 環境変数への直書き禁止。Secret Manager を使い、Cloud Run から参照する
- マウント方法は2種類：環境変数としてマウント or ファイルとしてマウント
- ローテーション時は Secret のバージョンを更新 → Cloud Run の新リビジョンへ反映

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| `--allow-unauthenticated` を全サービスに設定 | 公開 API のみ許可。内部 API は IAM で制限 |
| デフォルトのサービスアカウント (`compute@...`) を使う | サービスごとに専用 SA を作成 |
| DATABASE_URL を環境変数に平文で設定 | Secret Manager に保存し参照 |
| サービス間通信で Firebase JWT を使う | GCP の OIDC ID Token を使う |
| SA に `roles/editor` を付与 | 必要最小のロールのみ |

---

## コード / 設計例

### サービス間通信（FastAPI から別の Cloud Run を呼ぶ）

```python
import google.auth.transport.requests
import google.oauth2.id_token

async def call_internal_service(url: str) -> dict:
    auth_req = google.auth.transport.requests.Request()
    token = google.oauth2.id_token.fetch_id_token(auth_req, audience=url)
    async with httpx.AsyncClient() as client:
        resp = await client.get(
            url,
            headers={"Authorization": f"Bearer {token}"},
        )
    resp.raise_for_status()
    return resp.json()
```

### Terraform でサービスアカウントと IAM を最小権限で設定

```hcl
resource "google_service_account" "api" {
  account_id = "cloud-run-api"
}

resource "google_secret_manager_secret_iam_member" "db_url" {
  secret_id = google_secret_manager_secret.db_url.id
  role      = "roles/secretmanager.secretAccessor"
  member    = "serviceAccount:${google_service_account.api.email}"
}

resource "google_cloud_run_v2_service_iam_member" "internal_invoker" {
  name   = google_cloud_run_v2_service.internal.name
  role   = "roles/run.invoker"
  member = "serviceAccount:${google_service_account.api.email}"
}
```

---

## トレードオフ

| 観点 | 選択肢 A | 選択肢 B |
|---|---|---|
| 内部 API の公開制御 | IAM Invoker（GCP ネイティブ） | API Gateway + APIキー |
| シークレット参照 | Secret Manager マウント（推奨） | 環境変数直書き（NG） |
| ローカル開発 | ADC（Application Default Credentials）で代替 | サービスアカウントキー（漏洩リスク高） |
| サービス間認証 | OIDC ID Token（GCP 管理、期限あり） | 共有 API キー（管理コスト高） |

**ADC（Application Default Credentials）**: ローカル開発時は `gcloud auth application-default login` で認証。本番と同じコードが動く。

---

## チェックリスト

- [ ] 内部サービスに `--allow-unauthenticated` が設定されていないか確認
- [ ] 各 Cloud Run サービスに専用のサービスアカウントを割り当て済みか
- [ ] DATABASE_URL / API シークレットは Secret Manager 経由で参照しているか
- [ ] サービス間通信は OIDC ID Token で認証しているか
- [ ] SA に付与されているロールは最小権限か（`roles/editor` 等の広いロールがないか）
