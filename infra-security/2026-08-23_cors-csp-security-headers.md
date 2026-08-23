# CORS・CSP・セキュリティヘッダーの正しい設定

## 概要

Webアプリケーションには「コードのバグ」以外に「HTTPヘッダーの設定ミス」による脆弱性が多数存在する。
CORS・CSP・各種セキュリティヘッダーは正しく設定することでXSS・クリックジャッキング・情報漏洩を防ぐ。
FastAPI + Cloud Run 構成では、どこでヘッダーを付与するかの設計判断も重要。

---

## 仕組みの要点

### CORS（Cross-Origin Resource Sharing）
- ブラウザが「異なるオリジンへのリクエスト」を制御する仕組み
- `Access-Control-Allow-Origin` で許可するオリジンを指定
- `credentials: true` のときはワイルドカード `*` 不可、明示的なオリジン指定が必須
- プリフライトリクエスト（OPTIONS）は実リクエスト前に飛ぶ

### CSP（Content Security Policy）
- ブラウザが読み込めるリソースの出所を制限する
- XSS攻撃でインジェクトされたスクリプトの実行をブロック
- `Content-Security-Policy` ヘッダーで宣言的に制御
- `Report-Only` モードで違反をログに流してから段階的に有効化できる

### 主要セキュリティヘッダー一覧
| ヘッダー | 役割 |
|---|---|
| `Strict-Transport-Security` | HTTPS強制（HSTS） |
| `X-Frame-Options` | クリックジャッキング防止 |
| `X-Content-Type-Options` | MIMEスニッフィング防止 |
| `Referrer-Policy` | リファラー情報の制御 |
| `Permissions-Policy` | カメラ・位置情報等の権限制御 |

---

## アンチパターン vs 正しい設計

### ❌ アンチパターン
- `Access-Control-Allow-Origin: *` + `credentials: true` → ブラウザエラー or 認証情報漏洩
- CSP未設定 → XSSが刺さったときスクリプトが自由に実行される
- `X-Frame-Options` 未設定 → クリックジャッキング攻撃が成立
- 開発環境の CORS 設定を本番にそのまま流用（`allow_origins=["*"]`）

### ✅ 正しい設計
- 許可オリジンは環境変数で管理し、本番・ステージング・ローカルで分離
- CSPは `Report-Only` → 違反確認 → `enforce` の順で段階導入
- ヘッダーは Cloud Run の前段（Cloudflare or API Gateway）で一元管理
- セキュリティヘッダーをミドルウェアで統一して漏れを防ぐ

---

## コード/設計例（FastAPI）

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
import os

app = FastAPI()

ALLOWED_ORIGINS = os.getenv("ALLOWED_ORIGINS", "").split(",")

app.add_middleware(
    CORSMiddleware,
    allow_origins=ALLOWED_ORIGINS,
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
)

class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)
        response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
        response.headers["Content-Security-Policy"] = (
            "default-src 'self'; script-src 'self'; object-src 'none'"
        )
        return response

app.add_middleware(SecurityHeadersMiddleware)
```

---

## トレードオフ

| 判断 | メリット | デメリット |
|---|---|---|
| CSP厳格化（`script-src 'self'`のみ） | XSS耐性が高い | インラインスクリプト・CDN全禁止で実装制約大 |
| CSP Report-Only 運用 | 段階導入で安全 | 完全施行まで保護されない期間がある |
| ヘッダーをCloudflareで設定 | アプリ変更不要、一元管理 | ローカル開発時に再現できない場合がある |
| アプリ側ミドルウェアで設定 | ローカルでも同じ動作 | 全APIサービスに実装が必要 |

**推奨**: ヘッダーはCloudflare（Transform Rules）で基本設定 + アプリ側で補完。CORSはアプリ側で制御（オリジンがアプリロジックに依存するため）。

---

## チェックリスト

- [ ] `ALLOWED_ORIGINS` は環境変数で管理し、本番では具体的なドメインのみ許可している
- [ ] `credentials: true` の場合、`allow_origins=["*"]` になっていない
- [ ] `Strict-Transport-Security` ヘッダーが全レスポンスに付与されている
- [ ] CSPを設定し、少なくとも `Report-Only` で違反ログを収集している
- [ ] `X-Frame-Options: DENY` または CSPの `frame-ancestors 'none'` でクリックジャッキング対策済み
