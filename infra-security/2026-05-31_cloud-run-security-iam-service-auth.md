# Cloud Run のセキュリティ設定・IAM・サービス間認証

## 概要

Cloud Run はサーバーレスでコンテナを動かす GCP サービス。  
「動くだけ」の設定は致命的な穴を抱えやすい。  
- デフォルトは「誰でもアクセス可能」になる場合がある
- サービスアカウントの権限を絞らないと、侵害時の被害が全リソースに波及する
- サービス間通信を平文・無検証にすると内部ネットワークも信頼できない

FastAPI + Neon + Firebase Auth スタックでも、**Cloud Run の IAM 設定が最初の防衛線**になる。

---

## 仕組みの要点

### Cloud Run の認証モデル

- **未認証アクセス許可 (allUsers)**: 公開 API に使用。Firebase Auth の JWT を自前で検証する必要がある
- **認証済みアクセスのみ (IAM)**: サービス間通信やバックエンド専用に使用。Google ID Token で認証
- IAM ロール `roles/run.invoker` を付与したサービスアカウントのみ呼び出し可能

### サービスアカウントの最小権限原則

| サービス | 必要な権限 | アンチパターン |
|---|---|---|
| FastAPI (Cloud Run) | Neon への接続のみ | Editor / Owner 権限 |
| バッチワーカー | Cloud Storage 読み書き | プロジェクト全体の権限 |
| CI/CD | 特定のサービスへの deploy のみ | Owner 権限の SA キー |

### サービス間認証フロー

```
Service A (呼び出し元)
  └─ メタデータサーバーから ID Token 取得
       └─ Authorization: Bearer <id_token> でリクエスト
            └─ Service B (Cloud Run) が Google で自動検証
```

- ID Token の audience は呼び出し先の URL を指定する（再利用攻撃を防ぐ）
- サービスアカウントキー（JSON ファイル）は原則不要・使わない

---

## アンチパターン vs 正しい設計

### アンチパターン

- `allUsers` に `run.invoker` を付与 → 誰でもバックエンドを叩ける
- デフォルトのコンピュートサービスアカウントを使用 → 不要な権限を多数保有
- サービスアカウントキーをコードや環境変数に埋め込む → 漏洩リスク
- `--allow-unauthenticated` を全サービスに設定 → 内部 API も公開される

### 正しい設計

- 公開エンドポイント：`--allow-unauthenticated` + アプリ層で Firebase JWT 検証
- 内部エンドポイント：IAM 認証のみ（`--no-allow-unauthenticated`）
- サービスごとに専用 SA を作成し、最小権限を付与
- SA キーは使わず、Workload Identity / メタデータサーバーを使う

---

## コード/設計例

### FastAPI での Firebase JWT 検証（公開エンドポイント）

```python
from firebase_admin import auth, credentials, initialize_app
from fastapi import HTTPException, Security
from fastapi.security import HTTPBearer

initialize_app(credentials.ApplicationDefault())
bearer = HTTPBearer()

async def verify_firebase_token(token=Security(bearer)):
    try:
        decoded = auth.verify_id_token(token.credentials)
        return decoded
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")
```

### サービス間通信（Cloud Run → Cloud Run）

```python
import google.auth.transport.requests
import google.oauth2.id_token
import httpx

async def call_internal_service(url: str, payload: dict):
    audience = "https://internal-service-xxxx-an.a.run.app"
    token = google.oauth2.id_token.fetch_id_token(
        google.auth.transport.requests.Request(), audience
    )
    async with httpx.AsyncClient() as client:
        return await client.post(
            url, json=payload,
            headers={"Authorization": f"Bearer {token}"}
        )
```

---

## トレードオフ

| 選択肢 | メリット | デメリット |
|---|---|---|
| Firebase JWT を自前検証 | 柔軟なクレーム検査が可能 | 公開鍵の取得・キャッシュが必要 |
| Cloud Run IAM 認証 | Google が全自動で検証 | 外部クライアントには使えない |
| SA キーファイル | どこでも動く | 漏洩時の被害大・ローテーション負荷 |
| Workload Identity | キーレスで安全 | GCP 外からは使えない |

---

## チェックリスト

- [ ] 公開 API と内部 API でサービスを分離し、内部は `--no-allow-unauthenticated` を設定している
- [ ] 各 Cloud Run サービスに専用のサービスアカウントを割り当て、Editor/Owner を使っていない
- [ ] サービスアカウントキー（JSON）をコード・環境変数・リポジトリに置いていない
- [ ] サービス間通信では audience を呼び出し先 URL に固定した ID Token を使っている
- [ ] Firebase JWT 検証は公開鍵をキャッシュして毎リクエスト外部取得しない実装になっている
