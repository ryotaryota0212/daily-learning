# Cloud Run のセキュリティ設定・IAM・サービス間認証

## 概要

Cloud Run はサーバーレスコンテナ実行環境として手軽にスケールするが、誤った設定では誰でも叩けるパブリックAPIや、過剰権限のサービスアカウントが生まれやすい。
「動けば良い」から「安全に動く」に移行するには IAM の最小権限・サービス間認証・シークレット管理の3点が核となる。
FastAPI + Cloud Run 構成でこれを正しく設計できることが、AI時代のシステム設計力の基礎となる。

---

## 仕組みの要点

### Cloud Run のアイデンティティモデル
- **サービスアカウント（SA）**：各 Cloud Run サービスが持つ実行 ID。これに IAM ロールを付与して権限を制御する
- **未認証アクセス**：Cloud Run は「全員に公開」か「IAM で認証済みのみ」かを選択できる
- **Audience（対象者）**：サービス間通信では OIDC トークンの `aud` フィールドで呼び出し先を特定する

### サービス間認証（OIDC フロー）
1. 呼び出し元 SA が Google に OIDC トークンを要求（`aud` = 呼び出し先 URL）
2. トークンを `Authorization: Bearer <token>` に付与してリクエスト
3. Cloud Run が Google の公開鍵でトークン検証 → 許可 or 拒否

### IAM ロールの使い分け
| ロール | 用途 |
|---|---|
| `roles/run.invoker` | Cloud Run を呼び出す権限 |
| `roles/cloudsql.client` | Cloud SQL への接続 |
| `roles/secretmanager.secretAccessor` | シークレットの読み取り |
| `roles/storage.objectViewer` | GCS バケット読み取り |

---

## アンチパターン vs 正しい設計

### アンチパターン
- `--allow-unauthenticated` を内部 API にも付ける（誰でも叩ける）
- デフォルト SA（`PROJECT_NUMBER-compute@...`）を使う（他サービスと権限が混在）
- 環境変数にシークレットを直書き（`SECRET=abc123`）
- SA に `roles/owner` や `roles/editor` を付与（過剰権限）

### 正しい設計
- 外部公開 API のみ `--allow-unauthenticated`、内部 API は IAM 認証
- サービスごとに専用 SA を作成し、必要最小限のロールだけ付与
- シークレットは Secret Manager に保存し、SA に `secretAccessor` のみ付与
- VPC Service Controls で境界を設けてネットワーク分離

---

## コード/設計例

### デプロイ時の設定（最小権限）
```bash
# 専用サービスアカウント作成
gcloud iam service-accounts create fastapi-sa \
  --display-name="FastAPI Cloud Run SA"

# 必要ロールのみ付与
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:fastapi-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# 内部APIは認証必須でデプロイ
gcloud run deploy fastapi-internal \
  --service-account=fastapi-sa@PROJECT_ID.iam.gserviceaccount.com \
  --no-allow-unauthenticated
```

### FastAPI でサービス間認証トークンを取得
```python
import google.auth.transport.requests
import google.oauth2.id_token

def get_id_token(audience: str) -> str:
    request = google.auth.transport.requests.Request()
    token = google.oauth2.id_token.fetch_id_token(request, audience)
    return token

# 呼び出し元サービスから内部APIを叩く
import httpx

async def call_internal_api(url: str) -> dict:
    token = get_id_token(audience=url)
    async with httpx.AsyncClient() as client:
        resp = await client.get(
            url,
            headers={"Authorization": f"Bearer {token}"}
        )
    return resp.json()
```

### Secret Manager からの安全な読み込み
```python
from google.cloud import secretmanager

def get_secret(secret_id: str, project_id: str) -> str:
    client = secretmanager.SecretManagerServiceClient()
    name = f"projects/{project_id}/secrets/{secret_id}/versions/latest"
    response = client.access_secret_version(request={"name": name})
    return response.payload.data.decode("UTF-8")
```

---

## トレードオフ

| 観点 | 厳格な IAM 設定 | 緩い設定 |
|---|---|---|
| セキュリティ | 侵害範囲を最小化 | 1つの穴が全体に波及 |
| 開発速度 | SA 管理の手間がかかる | 初速は早い |
| デバッグ | 権限エラーの原因追跡が必要 | エラーが出にくい |
| コスト | Secret Manager は微量のコスト | 実質無料 |

**推奨**: 開発初期からサービスごとに SA を分けておく。後から分離するのはリスクが高い。

---

## チェックリスト

- [ ] Cloud Run サービスごとに専用サービスアカウントを作成し、最小権限を付与している
- [ ] 内部 API エンドポイントは `--no-allow-unauthenticated` で保護されている
- [ ] データベース接続文字列・API キー等のシークレットは Secret Manager で管理している
- [ ] サービス間通信では OIDC トークン（`google.oauth2.id_token`）を使っている
- [ ] Cloud Audit Logs でサービスアカウントの操作ログを記録・監視している
