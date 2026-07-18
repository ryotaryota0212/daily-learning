# Cloud Run のセキュリティ設定・IAM・サービス間認証

## 概要

Cloud Run はサーバーレスコンテナ実行環境だが、デフォルト設定のまま使うと認証なしでの公開アクセスや過剰権限などリスクがある。
FastAPI + Firebase Auth + Neon のスタックでは、Cloud Run のIAM設計がセキュリティの要になる。
「誰が・どのサービスに・何の権限で」アクセスできるかを最小権限原則で設計することが重要。

## 仕組みの要点

### 認証の3層構造

- **外部ユーザー → Cloud Run**: Firebase Auth の IDトークンを検証
- **Cloud Run → Cloud Run（サービス間）**: Google-signed ID Token（OIDC）を使用
- **Cloud Run → GCPリソース（DB、GCS等）**: サービスアカウントのIAMロールで制御

### サービスアカウント設計

- 各Cloud Runサービスに専用のサービスアカウントを作成（デフォルトSAは使わない）
- 付与するロールは必要最小限（例: Cloud SQL Client のみ、Pub/Sub Publisher のみ）
- サービス間通信では呼び出し元SAに `roles/run.invoker` を付与

### ネットワークセキュリティ

- 内部サービスは `--ingress=internal` で外部アクセスを遮断
- VPC Connector でNeon（PostgreSQL）との通信をプライベートネットワーク経由に
- Cloud Armorと組み合わせてWAFを適用

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| `--allow-unauthenticated` をすべてのサービスに設定 | 公開エンドポイントのみに限定、他は内部通信 |
| デフォルトのCompute Engine SAを使用 | サービスごとに最小権限SAを作成 |
| サービス間でAPIキーを共有 | OIDCトークンによる署名付き認証 |
| DB接続情報を環境変数に平文で設定 | Secret Manager 経由で取得 |

## コード/設計例

```python
# FastAPI: サービス間呼び出し（OIDC認証付き）
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
    return resp.json()
```

```bash
# IAM設定（最小権限）
gcloud run deploy api-service \
  --service-account api-sa@project.iam.gserviceaccount.com \
  --ingress=internal-and-cloud-load-balancing \
  --no-allow-unauthenticated

# サービス間認証の許可
gcloud run services add-iam-policy-binding backend-service \
  --member="serviceAccount:api-sa@project.iam.gserviceaccount.com" \
  --role="roles/run.invoker"
```

## トレードオフ

| 観点 | 内容 |
|---|---|
| セキュリティ vs 開発速度 | IAMを細かく設定するほど安全だが、権限エラーのデバッグに時間がかかる |
| 内部通信 vs VPC | VPC Connectorはコスト増・設定複雑化、ただし接続の安定性が上がる |
| Secret Manager vs 環境変数 | Secret Managerはバージョン管理・監査ログが取れるが実装コストあり |

## チェックリスト

- [ ] 各Cloud Runサービスに専用サービスアカウントを作成したか
- [ ] `--allow-unauthenticated` は公開エンドポイントのみに限定されているか
- [ ] サービス間通信はOIDCトークンで認証されているか
- [ ] DB接続情報はSecret Manager経由で取得しているか
- [ ] IAMロールは最小権限（Custom Role or 事前定義ロールの絞り込み）になっているか
