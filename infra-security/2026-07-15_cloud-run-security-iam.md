# Cloud Run のセキュリティ設定・IAM・サービス間認証

## 概要

Cloud Run はサーバーレスでコンテナを実行できる GCP サービス。デフォルト設定のまま使うと意図せず公開状態になったり、サービス間の認証が甘くなりやすい。IAM・サービスアカウント・OIDC トークンの仕組みを正しく理解することで、最小権限・ゼロトラストな構成が実現できる。FastAPI + Firebase Auth スタックでは「ユーザー認証（Firebase）」と「サービス間認証（IAM）」を明確に分離することが重要。

---

## 仕組みの要点

### サービスアカウント（SA）と IAM ロール
- Cloud Run サービスは SA として動作し、SA に付与されたロールが権限の境界になる
- デフォルト SA（Compute Engine SA）は過剰な権限を持つ → 専用 SA を作成すること
- 代表的なロール:
  - `roles/run.invoker` — Cloud Run サービスを呼び出す権限
  - `roles/cloudsql.client` — Cloud SQL/Neon プロキシ経由の DB 接続
  - `roles/secretmanager.secretAccessor` — Secret Manager の読み取り

### サービス間認証（OIDC Identity Token）
- Cloud Run の「未認証呼び出しを禁止」設定で内部 API を保護
- 呼び出し元は **OIDC Identity Token** を取得し `Authorization: Bearer <token>` で渡す
- GCP メタデータサーバーがトークンを自動発行・検証するため、API Key 管理が不要

### Firebase Auth との役割分担
| 認証種別 | 使用場所 | トークン種別 |
|---|---|---|
| エンドユーザー認証 | クライアント → API Gateway | Firebase ID Token |
| サービス間認証 | API → 内部サービス | GCP OIDC Token |

この 2 つを混同すると「外部ユーザーが内部 API を呼べる」「内部間通信でも Firebase 検証が走る」などの設計ミスが起きる。

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| 内部 API に `--allow-unauthenticated` を付与 | 内部 API は `--no-allow-unauthenticated` を必須に |
| デフォルト SA をすべてのサービスで使い回す | サービスごとに専用 SA を作成・最小ロールのみ付与 |
| サービス間通信に共有 API Key を使う | OIDC Token を使う（GCP が署名検証・失効管理） |
| 本番・開発で同じ SA を使う | 環境ごとに SA を分離しロールを絞る |
| Ingress 制限なしで全公開 | `--ingress internal-and-cloud-load-balancing` で制限 |

---

## コード/設計例

```python
# FastAPI: 内部 Cloud Run サービスへの認証付きリクエスト
import google.auth.transport.requests
import google.oauth2.id_token
import httpx

async def call_internal_service(url: str) -> dict:
    auth_req = google.auth.transport.requests.Request()
    # audience = 呼び出し先の Cloud Run URL
    token = google.oauth2.id_token.fetch_id_token(auth_req, url)
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, headers={"Authorization": f"Bearer {token}"})
    resp.raise_for_status()
    return resp.json()
```

```bash
# デプロイ時のセキュリティ設定（gcloud）
gcloud run deploy my-service \
  --service-account my-service-sa@project.iam.gserviceaccount.com \
  --no-allow-unauthenticated \
  --ingress internal-and-cloud-load-balancing \
  --region asia-northeast1
```

---

## トレードオフ

| 設定 | メリット | デメリット |
|---|---|---|
| `--no-allow-unauthenticated` | 未認証アクセスをブロック | ローカル開発・curl テストが面倒になる |
| SA per service | 侵害時の影響範囲を限定 | SA 管理・ロール付与の手間が増える |
| Ingress: internal-only | 内部ネットワーク限定でセキュア | Cloud Shell や外部デバッグが不可 |
| OIDC Token | 鍵管理不要・自動失効 | ローカル環境での再現が難しい |

---

## チェックリスト

- [ ] 内部 API サービスに `--no-allow-unauthenticated` を設定している
- [ ] サービスごとに専用サービスアカウントを作成し、最小ロールのみ付与している
- [ ] サービス間通信に OIDC Identity Token を使っている（API Key 不使用）
- [ ] Firebase ID Token と GCP OIDC Token の検証ロジックを明確に分離している
- [ ] Ingress を `internal-and-cloud-load-balancing` 以下に制限している
