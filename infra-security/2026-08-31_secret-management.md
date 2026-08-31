# シークレット管理：環境変数・Secret Manager・Vault

## 概要

APIキー・DB接続文字列・JWT秘密鍵などの機密情報を安全に管理する方法はインフラ設計の根幹。
コードにハードコードしたり `.env` をリポジトリにコミットするだけで、全てが漏洩する。
Cloud Run + FastAPI の構成では「どこに置くか」「誰がアクセスできるか」「いつローテーションするか」の3点が設計の核となる。

---

## 仕組みの要点

**シークレットの管理レイヤー**
- **環境変数（最低限）**: 値はメモリ上のみ。コードに混入しない。ローカル開発向け `.env` はgitignore必須
- **Google Secret Manager**: GCP の秘匿情報ストア。バージョン管理・自動ローテーション・IAMで細粒度アクセス制御
- **HashiCorp Vault**: 自己ホスト型の高機能シークレット管理。動的シークレット（使い捨てDB認証）が強み
- **Cloud Run との統合**: サービスアカウントに `secretmanager.secretAccessor` ロールを付与し、起動時に注入

**アクセス制御の原則**
- 最小権限：各サービスが必要なシークレットのみ読める
- サービスアカウント単位でIAMバインディングを設定
- 人間のアクセスは監査ログ付きで制限

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| `.env` をgitにコミット | `.gitignore` + Secret Managerに移行 |
| コードにAPIキーをハードコード | 環境変数 or Secret Manager経由で取得 |
| 全サービスが同じシークレットを共有 | サービスごとに個別シークレット |
| シークレットのローテーションなし | 90日以内の自動ローテーション |
| ログにシークレット値が出力される | シークレットをログ除外・マスキング |

---

## コード/設計例

```python
# FastAPI + Cloud Run でのSecret Manager統合
from google.cloud import secretmanager
import os

def get_secret(secret_id: str) -> str:
    client = secretmanager.SecretManagerServiceClient()
    project = os.environ["GOOGLE_CLOUD_PROJECT"]
    name = f"projects/{project}/secrets/{secret_id}/versions/latest"
    response = client.access_secret_version(request={"name": name})
    return response.payload.data.decode("UTF-8")

# 起動時に1回だけ取得してキャッシュ（毎リクエストは非効率）
class Settings:
    def __init__(self):
        self.db_url = get_secret("neon-db-url")
        self.firebase_key = get_secret("firebase-service-account")

settings = Settings()  # アプリ起動時に初期化
```

**Cloud Run デプロイ時のIAM設定（terraform例）**
```hcl
resource "google_secret_manager_secret_iam_member" "run_accessor" {
  secret_id = google_secret_manager_secret.db_url.secret_id
  role      = "roles/secretmanager.secretAccessor"
  member    = "serviceAccount:${google_service_account.run_sa.email}"
}
```

---

## トレードオフ

| 手法 | メリット | デメリット |
|---|---|---|
| 環境変数のみ | シンプル・速い | バージョン管理なし、ローテーション手動 |
| Secret Manager | GCP統合・IAM・監査ログ | GCP依存、API呼び出しコスト |
| Vault | マルチクラウド・動的シークレット | 運用コスト高、自己管理が必要 |

**FastAPI + Cloud Run の推奨**: 起動時に Secret Manager から取得してメモリキャッシュ。ローテーション時は Cloud Run の新リビジョンをデプロイ。

---

## チェックリスト

- [ ] `.env` ファイルが `.gitignore` に含まれているか
- [ ] Cloud Run サービスアカウントに最小権限のみ付与されているか
- [ ] シークレットのバージョン管理と90日ローテーション計画があるか
- [ ] シークレット値がログ・エラーメッセージに出力されないか
- [ ] ローカル開発用と本番用でシークレットが分離されているか
