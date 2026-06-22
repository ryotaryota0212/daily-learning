# Cloud Run のセキュリティ設定・IAM・サービス間認証

## 概要

Cloud Run はサーバーレスコンテナ基盤として手軽だが、デフォルト設定のまま使うと「誰でもアクセスできるAPI」や「最強権限で動くサービス」が量産される。
FastAPI + Firebase Auth + Neon のスタックでは、Cloud Run の IAM・ネットワーク・サービス間認証の設計がシステム全体のセキュリティ境界を決める。
「とりあえず動く」から「壊れにくく侵入されにくい」構成へ変えるための設計知識。

---

## 仕組みの要点

### IAM の2層構造
- **呼び出しレベル**: `roles/run.invoker` を持つ主体だけがエンドポイントを叩ける
- **サービスアカウントレベル**: Cloud Run 自体が持つ実行 SA の権限（DB・Secret・他サービス）

### サービス間認証（Service-to-Service）
1. 呼び出し元 SA に `roles/run.invoker` を付与
2. 呼び出し元が **OIDC トークン**を取得（`audience` = 呼び出し先 URL）
3. 呼び出し先が Google が発行した JWT を自動検証（--no-allow-unauthenticated 設定時）

### ネットワーク境界
- **Ingress**: `internal` or `internal-and-cloud-load-balancing` にすると直接アクセス不可
- **VPC Connector**: Neon (PostgreSQL) への接続は VPC 経由を推奨
- **Secret Manager**: 環境変数直書きではなくマウント参照を使う

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| `--allow-unauthenticated` で全公開 | ロードバランサー経由のみ許可、直接呼び出しは IAM で制限 |
| デフォルト SA（Compute Engine SA）を使用 | サービス専用 SA を作り最小権限を付与 |
| DB 接続文字列を env var に直書き | Secret Manager に格納し Cloud Run からマウント |
| 全サービスを同じ SA で実行 | サービスごとに SA を分離（侵害時の爆発半径を限定） |
| ソースコードに `GOOGLE_APPLICATION_CREDENTIALS` | Cloud Run 上では不要（自動で SA トークンを取得） |

---

## コード/設計例

### FastAPI からサービス間認証付きリクエスト

```python
import google.auth.transport.requests
import google.oauth2.id_token
import httpx

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

### Terraform での最小権限 SA 設定

```hcl
resource "google_service_account" "api_sa" {
  account_id = "fastapi-service"
}

resource "google_secret_manager_secret_iam_member" "db_url" {
  secret_id = google_secret_manager_secret.db_url.secret_id
  role      = "roles/secretmanager.secretAccessor"
  member    = "serviceAccount:${google_service_account.api_sa.email}"
}
```

---

## トレードオフ

| 選択肢 | メリット | デメリット |
|---|---|---|
| `--allow-unauthenticated` | 実装が簡単 | IAM 外からも叩ける、Firebase Auth だけが頼り |
| IAM 認証 + LB | 多層防御、攻撃面が狭い | LB コストが増加、設定が複雑 |
| 全サービス同一 SA | 管理が楽 | 1つ侵害されると全サービスへ横展開される |
| サービスごと SA 分離 | 爆発半径を限定できる | SA 数が増え管理コスト増 |

**推奨**: フロントエンドは Firebase Auth → LB → Cloud Run（unauthenticated OK）、バックエンド内部呼び出しは全て IAM + OIDC トークンで統一する。

---

## チェックリスト

- [ ] 本番 Cloud Run はすべて `--no-allow-unauthenticated` か内部 Ingress に設定されているか
- [ ] 各サービスが専用 SA を持ち、不要なロールが付いていないか（最小権限）
- [ ] DB 接続文字列・API キーは Secret Manager に格納し、env var 直書きを排除しているか
- [ ] サービス間通信は OIDC トークン認証を使い、ハードコードしたトークンが存在しないか
- [ ] IAM バインディングを定期的にレビューし、不要な invoker 権限が残っていないか
