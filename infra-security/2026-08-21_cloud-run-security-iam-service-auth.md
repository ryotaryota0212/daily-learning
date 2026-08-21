# Cloud Run のセキュリティ設定・IAM・サービス間認証

## 概要

Cloud Run はサーバーレスでコンテナを動かせる強力なプラットフォームだが、
デフォルト設定のまま使うと認証なし公開・過剰な権限・サービス間通信の平文化などのリスクがある。
FastAPI + Firebase Auth のスタックでも「Cloud Run をどう守るか」を理解しないと、
アプリ層のセキュリティだけでは意味がない。
AI時代でも「インフラの認証・認可設計」はコードで自動化できない判断領域であり、設計力が直接問われる。

---

## 仕組みの要点

### IAM の基本構造
- **サービスアカウント (SA)** : Cloud Run サービスが持つ「実行ID」。これに権限を付与する
- **roles/run.invoker** : Cloud Run を呼び出せる権限。外部公開か内部限定かの境界になる
- **最小権限の原則** : SA には必要な権限だけ。`roles/editor` や `roles/owner` は絶対にNG

### 認証パターン3種

| パターン | 用途 | 仕組み |
|---|---|---|
| Firebase Auth → Cloud Run | フロントエンド→API | Firebase ID Token をヘッダーに付けてAPIが検証 |
| Service-to-Service | API→内部サービス | OIDC トークン（Google 署名）をヘッダーに付与 |
| Cloud Run → Cloud SQL / Secrets | DBアクセス | IAM認証 + Secret Manager |

### ネットワーク境界
- **Ingress 設定** : `all`（全公開）/ `internal`（VPC内のみ）/ `internal-and-cloud-load-balancing`
- **VPC Connector** : Cloud Run → プライベートリソース（Cloud SQL, Memorystore等）への通信経路
- **Cloud Armor** : ロードバランサーに WAF ルールを適用して DDoS・SQLi をブロック

---

## アンチパターン vs 正しい設計

### アンチパターン
```
# NG: 全員が invoker になれる（認証なし公開）
gcloud run services add-iam-policy-binding my-api \
  --member="allUsers" --role="roles/run.invoker"

# NG: SA に過剰権限
--service-account=my-sa@project.iam.gserviceaccount.com
# my-sa が roles/editor を持っている
```

### 正しい設計
```
# OK: 認証必須（allUsers を付けない）
gcloud run services deploy my-api \
  --no-allow-unauthenticated \
  --service-account=my-api-sa@project.iam.gserviceaccount.com

# OK: 必要な権限だけ付与
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:my-api-sa@..." \
  --role="roles/secretmanager.secretAccessor"
```

---

## コード例: サービス間 OIDC 認証（FastAPI → 内部サービス呼び出し）

```python
import httpx
import google.auth.transport.requests
import google.oauth2.id_token

async def call_internal_service(url: str) -> dict:
    # Cloud Run が自動で OIDC トークンを取得
    auth_req = google.auth.transport.requests.Request()
    token = google.oauth2.id_token.fetch_id_token(auth_req, url)
    
    async with httpx.AsyncClient() as client:
        resp = await client.get(
            url,
            headers={"Authorization": f"Bearer {token}"},
            timeout=10.0,
        )
        resp.raise_for_status()
        return resp.json()
```

> ポイント: `fetch_id_token` は Cloud Run の実行環境では Metadata Server から自動取得。
> ローカル開発時は ADC（Application Default Credentials）を設定する。

---

## トレードオフ

| 選択 | メリット | デメリット |
|---|---|---|
| `--no-allow-unauthenticated` | 認証なしアクセスをインフラで遮断 | ヘルスチェックにも認証が必要（Cloud Load Balancer 経由で解決） |
| VPC Connector 経由 | プライベート通信、Cloud SQL 直接接続 | 月 $30〜のコスト、コールドスタート遅延増加 |
| Cloud Armor (WAF) | DDoS・L7攻撃をエッジで防御 | ルール管理コスト、誤検知リスク |
| Secret Manager | シークレットをコードに含めない | 呼び出しコスト（1万回$0.06）、レイテンシ増加 |

---

## チェックリスト

- [ ] Cloud Run サービスに `--no-allow-unauthenticated` が設定されているか
- [ ] 各サービスに専用 SA を作成し、`roles/editor` 等の過剰権限がないか
- [ ] サービス間通信は OIDC トークンを使っているか（APIキーの平文渡しになっていないか）
- [ ] DB接続文字列・APIキーは Secret Manager に格納し、SA 経由でアクセスしているか
- [ ] Ingress 設定は `all` のまま放置していないか（外部公開が必要なサービスのみ `all`）
