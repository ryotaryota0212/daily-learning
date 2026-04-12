# OWASP Top 10 脆弱性と対策 — FastAPI + Neon + Cloud Run + Firebase Auth スタック編

## 概要

OWASP Top 10 は、Webアプリケーションにおける最も重大なセキュリティリスクを定義した業界標準のリストである。
本ドキュメントでは、**FastAPI + PostgreSQL(Neon) + Cloud Run + Firebase Authentication** のスタックを前提に、
各脆弱性がどのように発生し、どう防御するかを具体的なコードとともに解説する。

対象バージョン: OWASP Top 10 — 2021 Edition

---

## 1. A01:2021 — Broken Access Control（アクセス制御の不備）

### どういう脆弱性か

認証済みのユーザーが、本来アクセスできないはずのリソース（他人のデータ、管理者機能など）にアクセスできてしまう問題。
最も発生頻度が高く、最もインパクトが大きい。

### なぜ起きるのか

- URLパラメータやリクエストボディの `user_id` をクライアントから受け取り、そのまま信用してしまう（IDOR: Insecure Direct Object Reference）
- エンドポイントに認可チェックが抜けている
- Row Level Security（RLS）が未設定でDBレベルの防御がない

### 具体的な攻撃例

```
# 攻撃者が他人の注文データを取得
GET /api/orders/12345
# 12345 は他のユーザーの注文ID
```

### FastAPI での対策

```python
from fastapi import Depends, HTTPException, status
from sqlalchemy.ext.asyncio import AsyncSession

async def get_order(
    order_id: int,
    current_user: User = Depends(get_current_user),  # Firebase Auth で認証
    db: AsyncSession = Depends(get_db),
):
    """注文取得 — 必ず所有者チェックを行う"""
    result = await db.execute(
        select(Order).where(
            Order.id == order_id,
            Order.user_id == current_user.uid,  # 所有者フィルタ（必須）
        )
    )
    order = result.scalar_one_or_none()
    if order is None:
        # 存在しない場合も権限がない場合も同じ 404 を返す
        # → 情報漏洩を防ぐ（403 だと「存在する」ことがバレる）
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND)
    return order
```

### Neon (PostgreSQL) RLS による多層防御

```sql
-- RLS を有効化
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- ユーザーは自分の注文のみ参照可能
CREATE POLICY orders_select_policy ON orders
    FOR SELECT
    USING (user_id = current_setting('app.current_user_id')::text);

-- アプリケーション側でセッションごとにユーザーIDを設定
-- SET LOCAL はトランザクション内でのみ有効（安全）
SET LOCAL app.current_user_id = 'firebase_uid_here';
```

**なぜ RLS も併用するのか**: アプリケーションコードのバグは必ず発生する。DBレベルでも防御することで、コードにアクセス制御の漏れがあっても被害を最小化できる（Defense in Depth）。

---

## 2. A02:2021 — Cryptographic Failures（暗号化の不備）

### どういう脆弱性か

機密データ（パスワード、個人情報、決済情報）が適切に暗号化されていない、または弱い暗号化が使われている問題。

### よくあるミス

- パスワードを平文やMD5/SHA1で保存している
- HTTPS を使っていない（Cloud Run はデフォルトで HTTPS なので安心）
- DB接続が暗号化されていない
- 機密データをログに出力している

### Neon への安全な接続

```python
# Neon はデフォルトで SSL 接続を強制するが、明示的に指定する
DATABASE_URL = "postgresql+asyncpg://user:pass@ep-xxx.neon.tech/dbname?ssl=require"

# SQLAlchemy の設定
engine = create_async_engine(
    DATABASE_URL,
    connect_args={
        "ssl": "require",  # TLS 接続を強制
    },
)
```

**なぜ `ssl=require` を明示するのか**: デフォルト設定に依存すると、環境やライブラリの更新で挙動が変わる可能性がある。明示的に指定することで意図を明確にする。

### 機密データのログ出力防止

```python
import logging
from pydantic import BaseModel, SecretStr

class UserCreate(BaseModel):
    email: str
    password: SecretStr  # ログに出力されても '**********' になる

# ログフィルタで機密情報をマスク
class SensitiveDataFilter(logging.Filter):
    SENSITIVE_PATTERNS = ["password", "token", "secret", "authorization"]

    def filter(self, record: logging.LogRecord) -> bool:
        message = record.getMessage().lower()
        for pattern in self.SENSITIVE_PATTERNS:
            if pattern in message:
                record.msg = "[REDACTED - sensitive data]"
                record.args = ()
        return True
```

---

## 3. A03:2021 — Injection（インジェクション）

### どういう脆弱性か

ユーザー入力がそのままクエリやコマンドの一部として解釈される問題。SQLインジェクションが最も有名。

### SQLインジェクションの例

```python
# 危険: 文字列連結でクエリを組み立てている
query = f"SELECT * FROM users WHERE email = '{email}'"
# email に ' OR '1'='1 を入れると全ユーザーが取得できる
```

### FastAPI + SQLAlchemy での安全な実装

```python
from sqlalchemy import select, text

# 安全: SQLAlchemy ORM を使う（パラメータが自動エスケープされる）
async def get_user_by_email(db: AsyncSession, email: str) -> User | None:
    result = await db.execute(
        select(User).where(User.email == email)
    )
    return result.scalar_one_or_none()

# 生SQL が必要な場合も必ずバインドパラメータを使う
async def search_users(db: AsyncSession, keyword: str):
    result = await db.execute(
        text("SELECT * FROM users WHERE name ILIKE :keyword"),
        {"keyword": f"%{keyword}%"},
    )
    return result.fetchall()
```

**なぜ ORM が安全なのか**: SQLAlchemy はすべてのパラメータをプリペアドステートメント（パラメータ化クエリ）として処理する。ユーザー入力が SQL の構文として解釈されることは原理的にない。

### NoSQLインジェクション（Firestore 使用時の注意）

```python
# Firestore はクエリ構造が固定的なので SQL インジェクションは起きにくいが、
# フィールド名をユーザー入力から組み立てると危険
field_name = request.query_params.get("sort_by")

# 危険: 任意のフィールドにアクセスできてしまう
db.collection("users").order_by(field_name)

# 安全: ホワイトリストで検証する
ALLOWED_SORT_FIELDS = {"name", "created_at", "email"}
if field_name not in ALLOWED_SORT_FIELDS:
    raise HTTPException(status_code=400, detail="Invalid sort field")
```

---

## 4. A04:2021 — Insecure Design（安全でない設計）

### どういう脆弱性か

実装のバグではなく、設計段階でセキュリティが考慮されていない問題。後からパッチを当てても根本解決にならない。

### よくあるミスと対策

| 安全でない設計 | 安全な設計 |
|---|---|
| パスワードリセットの質問が推測可能 | Firebase Auth のメールリンク認証を使う |
| レート制限なしのログイン試行 | レート制限 + アカウントロックアウト |
| 管理画面がインターネットに公開 | Cloud Run のイングレス制御で内部のみ |
| 決済処理がクライアント側で価格を決定 | サーバー側で価格を取得して決定 |

### Cloud Run のイングレス制御（管理API用）

```yaml
# cloud-run-admin-service.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: admin-api
  annotations:
    # 内部トラフィックのみ許可（インターネットからアクセス不可）
    run.googleapis.com/ingress: internal
```

**なぜイングレス制御が重要なのか**: 管理APIは認証があっても公開すべきではない。認証の脆弱性が見つかった場合のリスクを最小化するため、ネットワークレベルでもアクセスを制限する。

---

## 5. A05:2021 — Security Misconfiguration（セキュリティの設定ミス）

### どういう脆弱性か

デフォルト設定のまま本番運用、不要な機能の有効化、過剰な権限設定など。

### FastAPI での設定ミス防止

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
import os

app = FastAPI(
    # 本番では Swagger UI を無効化する
    docs_url="/docs" if os.getenv("ENV") != "production" else None,
    redoc_url="/redoc" if os.getenv("ENV") != "production" else None,
    openapi_url="/openapi.json" if os.getenv("ENV") != "production" else None,
)

# CORS: 許可するオリジンを明示的に指定する
# allow_origins=["*"] は絶対に本番で使わない
ALLOWED_ORIGINS = os.getenv("ALLOWED_ORIGINS", "").split(",")
app.add_middleware(
    CORSMiddleware,
    allow_origins=ALLOWED_ORIGINS,  # 例: ["https://myapp.com"]
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
)
```

### Cloud Run の最小権限 IAM

```bash
# サービスアカウントに最小限の権限のみ付与
gcloud iam service-accounts create api-service \
    --display-name="API Service Account"

# Cloud SQL (Neon) への接続権限のみ付与
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:api-service@$PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/cloudsql.client"

# Secret Manager へのアクセス権限（シークレット読み取りのみ）
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:api-service@$PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/secretmanager.secretAccessor"
```

**なぜ最小権限にするのか**: サービスアカウントが侵害された場合、攻撃者ができることを最小限に抑えるため。`roles/editor` のような広い権限は絶対に付与しない。

---

## 6. A06:2021 — Vulnerable and Outdated Components（脆弱で古いコンポーネント）

### どういう脆弱性か

既知の脆弱性を含むライブラリやフレームワークを使い続けている問題。

### 対策

```bash
# pip-audit で Python パッケージの脆弱性をスキャン
pip install pip-audit
pip-audit

# Dockerfile で軽量ベースイメージを使う（攻撃面の最小化）
# 危険: python:3.12 （フルイメージ、不要なツールが多数含まれる）
# 安全: python:3.12-slim
```

```dockerfile
# 安全な Dockerfile の例
FROM python:3.12-slim AS builder

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --target=/app/deps -r requirements.txt

FROM python:3.12-slim
# 非rootユーザーで実行
RUN useradd --create-home appuser
USER appuser

WORKDIR /app
COPY --from=builder /app/deps /app/deps
COPY . .

ENV PYTHONPATH=/app/deps
CMD ["python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

---

## 7. A07:2021 — Identification and Authentication Failures（認証の不備）

### どういう脆弱性か

認証メカニズムの実装ミスにより、攻撃者が他のユーザーになりすませる問題。

### Firebase Auth を使った安全な認証

```python
from firebase_admin import auth, initialize_app
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

initialize_app()
security = HTTPBearer()

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
) -> dict:
    """Firebase ID トークンを検証する"""
    token = credentials.credentials
    try:
        decoded_token = auth.verify_id_token(
            token,
            check_revoked=True,  # トークン無効化チェック（重要）
        )
    except auth.RevokedIdTokenError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token has been revoked",
        )
    except auth.InvalidIdTokenError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid token",
        )
    return decoded_token
```

**なぜ `check_revoked=True` が重要なのか**: ユーザーがパスワード変更やアカウント停止をした場合、既存のトークンを即座に無効化する必要がある。これを省略すると、漏洩したトークンがトークンの有効期限（デフォルト1時間）まで使い続けられてしまう。

### セッション管理の注意点

```python
# Firebase Auth のカスタムクレームで役割管理
# クライアント側で role を送信させるのは危険（改ざん可能）
async def require_admin(
    current_user: dict = Depends(get_current_user),
) -> dict:
    """管理者権限を要求する依存関数"""
    if not current_user.get("admin", False):
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Admin access required",
        )
    return current_user

# Firebase Admin SDK で管理者クレームを設定（サーバー側のみ）
# auth.set_custom_user_claims(uid, {"admin": True})
```

---

## 8. A08:2021 — Software and Data Integrity Failures（ソフトウェアとデータの整合性不備）

### どういう脆弱性か

コードやデータの整合性を検証せずに信頼してしまう問題。CI/CDパイプラインの汚染、安全でないデシリアライゼーションなど。

### Webhook の署名検証（Stripe の例）

```python
import stripe
from fastapi import Request, HTTPException

@app.post("/webhooks/stripe")
async def stripe_webhook(request: Request):
    payload = await request.body()
    sig_header = request.headers.get("stripe-signature")

    try:
        # 署名を検証してペイロードの改ざんを検出
        event = stripe.Webhook.construct_event(
            payload, sig_header, STRIPE_WEBHOOK_SECRET
        )
    except ValueError:
        raise HTTPException(status_code=400, detail="Invalid payload")
    except stripe.error.SignatureVerificationError:
        raise HTTPException(status_code=400, detail="Invalid signature")

    # 検証済みのイベントのみ処理
    if event["type"] == "checkout.session.completed":
        await handle_checkout_completed(event["data"]["object"])

    return {"status": "ok"}
```

**なぜ署名検証が必須なのか**: Webhook エンドポイントは公開URLなので、攻撃者が偽のイベントを送信できる。署名検証なしでは、偽の支払い完了通知でサービスを無料利用される可能性がある。

---

## 9. A09:2021 — Security Logging and Monitoring Failures（ログとモニタリングの不備）

### どういう脆弱性か

セキュリティイベントが記録されておらず、攻撃を検知できない、またはインシデント対応ができない問題。

### 構造化ログの実装

```python
import structlog
from fastapi import Request

# structlog で構造化ログを設定
structlog.configure(
    processors=[
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer(),
    ],
)
logger = structlog.get_logger()

# セキュリティイベントのログ記録ミドルウェア
@app.middleware("http")
async def security_logging_middleware(request: Request, call_next):
    response = await call_next(request)

    # 認証失敗をログに記録（不正アクセス試行の検知用）
    if response.status_code == 401:
        logger.warning(
            "authentication_failed",
            path=request.url.path,
            method=request.method,
            ip=request.client.host,
            user_agent=request.headers.get("user-agent"),
        )

    # 認可失敗をログに記録
    if response.status_code == 403:
        logger.warning(
            "authorization_failed",
            path=request.url.path,
            method=request.method,
            ip=request.client.host,
        )

    return response
```

### Cloud Run + Cloud Logging の活用

```python
# Cloud Run では stdout/stderr が自動的に Cloud Logging に送信される
# JSON形式で出力すると、Cloud Logging で構造化ログとして扱われる
import json
import sys

def log_security_event(event_type: str, details: dict):
    """Cloud Logging に構造化セキュリティイベントを送信"""
    log_entry = {
        "severity": "WARNING",
        "message": f"Security event: {event_type}",
        "event_type": event_type,
        **details,
    }
    print(json.dumps(log_entry), file=sys.stderr)
```

**なぜ構造化ログが重要なのか**: テキストログでは検索・集計が困難。JSON形式の構造化ログなら、Cloud Logging のクエリで「直近1時間に同一IPから10回以上認証失敗」といった分析が容易にできる。

---

## 10. A10:2021 — Server-Side Request Forgery (SSRF)

### どういう脆弱性か

サーバーがユーザー指定のURLにリクエストを送信する機能を悪用して、内部ネットワークのリソースにアクセスされる問題。

### 脆弱なコード例

```python
import httpx

# 危険: ユーザーが指定した URL にそのままリクエストを送信
@app.post("/api/fetch-url")
async def fetch_url(url: str):
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
    # 攻撃者が url=http://169.254.169.254/latest/meta-data/ を指定すると
    # クラウドのメタデータサービスにアクセスされ、認証情報が漏洩する
    return {"content": response.text}
```

### 安全な実装

```python
from urllib.parse import urlparse
import ipaddress

ALLOWED_SCHEMES = {"https"}
BLOCKED_NETWORKS = [
    ipaddress.ip_network("10.0.0.0/8"),
    ipaddress.ip_network("172.16.0.0/12"),
    ipaddress.ip_network("192.168.0.0/16"),
    ipaddress.ip_network("169.254.0.0/16"),  # リンクローカル（メタデータサービス）
    ipaddress.ip_network("127.0.0.0/8"),     # ループバック
]

def validate_url(url: str) -> str:
    """SSRF を防ぐための URL バリデーション"""
    parsed = urlparse(url)

    # スキームの検証
    if parsed.scheme not in ALLOWED_SCHEMES:
        raise ValueError(f"Scheme {parsed.scheme} is not allowed")

    # ホスト名を解決してIPアドレスを取得
    import socket
    try:
        ip = ipaddress.ip_address(socket.gethostbyname(parsed.hostname))
    except (socket.gaierror, ValueError):
        raise ValueError("Cannot resolve hostname")

    # プライベートIPへのアクセスをブロック
    for network in BLOCKED_NETWORKS:
        if ip in network:
            raise ValueError("Access to internal networks is not allowed")

    return url
```

**なぜ Cloud Run でも SSRF 対策が必要なのか**: Cloud Run はGCPの内部ネットワークに接続可能であり、メタデータサーバー（`169.254.169.254`）や他の内部サービスにアクセスできる。SSRFにより、サービスアカウントのアクセストークンが漏洩するリスクがある。

---

## 総合チェックリスト

### アクセス制御
- [ ] すべてのエンドポイントに認証・認可チェックがあるか
- [ ] IDOR 対策として、リソースアクセス時に所有者チェックをしているか
- [ ] Neon の RLS ポリシーが有効化されているか
- [ ] 管理APIは Cloud Run のイングレス制御で内部のみに制限しているか

### 暗号化
- [ ] Neon への接続で `ssl=require` を指定しているか
- [ ] 機密データがログに出力されていないか
- [ ] パスワードを自前で管理せず Firebase Auth に委譲しているか

### インジェクション
- [ ] SQLAlchemy ORM またはパラメータ化クエリを使用しているか
- [ ] ユーザー入力をSQL文字列に直接連結していないか
- [ ] フィールド名・テーブル名にユーザー入力を使う場合はホワイトリスト検証しているか

### 認証
- [ ] Firebase ID トークンの検証で `check_revoked=True` を指定しているか
- [ ] ユーザーの役割(role)はカスタムクレームで管理し、クライアント入力を信用していないか
- [ ] トークンの有効期限は適切か

### 設定
- [ ] 本番環境で Swagger UI / ReDoc を無効化しているか
- [ ] CORS の `allow_origins` が `*` になっていないか
- [ ] Cloud Run のサービスアカウントに最小権限のみ付与しているか
- [ ] Docker イメージで非root ユーザーを使用しているか

### 依存関係
- [ ] `pip-audit` でパッケージの脆弱性をスキャンしているか
- [ ] slim ベースイメージを使用しているか
- [ ] CI/CD で定期的に依存関係の更新チェックを行っているか

### Webhook / 外部連携
- [ ] Stripe 等の Webhook で署名検証を行っているか
- [ ] SSRF 対策として、外部URL取得時にプライベートIP をブロックしているか

### ログ・モニタリング
- [ ] 認証失敗・認可失敗をログに記録しているか
- [ ] ログは JSON 構造化形式で出力しているか
- [ ] Cloud Logging でセキュリティアラートを設定しているか

---

## 参考リンク

- OWASP Top 10 (2021): https://owasp.org/Top10/
- FastAPI Security: https://fastapi.tiangolo.com/tutorial/security/
- Firebase Admin Python SDK: https://firebase.google.com/docs/admin/setup
- Cloud Run Security: https://cloud.google.com/run/docs/securing/service-identity
- Neon Connection Security: https://neon.tech/docs/connect/connect-securely
