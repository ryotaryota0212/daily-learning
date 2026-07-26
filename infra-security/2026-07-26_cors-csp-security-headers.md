# CORS・CSP・セキュリティヘッダーの正しい設定

## 概要

ブラウザはWebアプリを守るセキュリティ機構をいくつか持っているが、**サーバー側が正しくヘッダーを返さないと機能しない**。
CORS・CSP・その他セキュリティヘッダーは「設定すれば安全」ではなく「誤った設定が逆に脆弱性になる」領域。
FastAPI + Firebase Auth + Cloud Run 構成でも同様で、デフォルト設定のまま本番投入するとXSS・情報漏洩リスクが残る。

---

## 仕組みの要点

### CORS（Cross-Origin Resource Sharing）
- ブラウザが「異なるオリジンへのリクエスト」を制限する Same-Origin Policy の緩和機構
- サーバーが `Access-Control-Allow-Origin` を返すことで許可するオリジンを宣言
- `OPTIONS` プリフライトリクエストへの応答も必要（`POST` / `PUT` / カスタムヘッダー使用時）
- **`*` は認証付きリクエスト（Cookie / Authorization）と共存不可**

### CSP（Content Security Policy）
- どのオリジンからスクリプト・スタイル・画像を読み込めるかをブラウザに指示
- XSS攻撃の被害を限定する「最後の防衛ライン」
- `Content-Security-Policy` ヘッダーまたは `<meta>` タグで設定
- 違反はブロックされ、`report-uri` でレポート収集も可能

### その他の重要ヘッダー

| ヘッダー | 役割 |
|---|---|
| `Strict-Transport-Security` | HTTPS強制（HSTS） |
| `X-Content-Type-Options: nosniff` | MIMEスニッフィング防止 |
| `X-Frame-Options: DENY` | クリックジャッキング防止 |
| `Referrer-Policy` | リファラー情報の漏洩制限 |
| `Permissions-Policy` | カメラ・位置情報等のAPI制限 |

---

## アンチパターン vs 正しい設計

### CORS

| アンチパターン | 正しい設計 |
|---|---|
| `Allow-Origin: *` を全エンドポイントに設定 | 許可オリジンをホワイトリストで管理 |
| 開発用の `localhost` を本番設定に混入 | 環境変数でオリジンを切り替え |
| `Allow-Methods: *` で全メソッドを許可 | 必要なメソッドのみ明示 |

### CSP

| アンチパターン | 正しい設計 |
|---|---|
| `unsafe-inline` / `unsafe-eval` を許可 | nonce or hash で特定スクリプトのみ許可 |
| CSPを設定しない | 最初はReportOnly→段階的に強化 |
| `default-src *` | `default-src 'none'` から始めて必要なものを追加 |

---

## コード例（FastAPI）

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request

app = FastAPI()

ALLOWED_ORIGINS = ["https://example.com", "https://app.example.com"]

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
        response.headers["Strict-Transport-Security"] = "max-age=63072000; includeSubDomains"
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
        response.headers["Content-Security-Policy"] = (
            "default-src 'none'; script-src 'self'; style-src 'self'; "
            "img-src 'self' data:; connect-src 'self' https://firebaseapp.com"
        )
        return response

app.add_middleware(SecurityHeadersMiddleware)
```

---

## トレードオフ

| 選択肢 | メリット | デメリット |
|---|---|---|
| CSP厳格化（nonce使用） | XSSリスク大幅低減 | CDN・外部ライブラリとの調整コスト増 |
| CORS広め許可 | 開発速度向上 | 意図しないオリジンからのAPI呼び出しリスク |
| HSTS有効化 | 中間者攻撃防止 | HTTP→HTTPS移行期に接続不能になるリスク |
| `Report-Only` モード | 段階的移行可能 | 実際にはブロックされないため保護効果なし |

---

## チェックリスト

- [ ] `Access-Control-Allow-Origin` は `*` でなくホワイトリスト管理になっているか
- [ ] CSPに `unsafe-inline` / `unsafe-eval` が含まれていないか
- [ ] `Strict-Transport-Security` が設定され、`max-age` が十分長いか（推奨: 2年）
- [ ] `X-Content-Type-Options: nosniff` と `X-Frame-Options: DENY` が設定されているか
- [ ] ステージング環境でCSP `Report-Only` を使い、違反を検知してから本番強化しているか
