# CORS・CSP・セキュリティヘッダーの正しい設定

## 概要

ブラウザはアプリを守るためにさまざまなセキュリティ機構を持つが、設定を誤ると「守れていないのに守れていると思い込む」状態になる。
CORS・CSP・各種セキュリティヘッダーは「開発者がサーバーからブラウザに指示を送り、ブラウザが代わりに防御する」仕組みである。
FastAPI + Cloud Run 構成でこれらを正しく理解し設定することが、XSS・CSRF・クリックジャッキング等を防ぐ前提となる。

---

## 仕組みの要点

### CORS（Cross-Origin Resource Sharing）

- 異なるオリジン（ドメイン・ポート・プロトコルが異なる）へのリクエストをブラウザが制限する仕組み
- サーバーが `Access-Control-Allow-Origin` 等のヘッダーで「許可するオリジン」を宣言
- Preflight（OPTIONSリクエスト）: PUT/DELETE/カスタムヘッダーを含むリクエスト前にブラウザが事前確認
- **CORS はブラウザの機能であり、サーバー間通信（curl等）には効かない**

### CSP（Content Security Policy）

- `Content-Security-Policy` ヘッダーで「どのオリジンのリソースをロードしてよいか」を指定
- XSS 攻撃でインジェクションされたスクリプトの実行をブラウザに拒否させる
- `report-uri` / `report-to` でポリシー違反を収集しながら段階的に厳格化できる

### その他の重要ヘッダー

| ヘッダー | 効果 |
|---|---|
| `Strict-Transport-Security` | HTTPS 強制（HSTS） |
| `X-Frame-Options` | クリックジャッキング防止 |
| `X-Content-Type-Options: nosniff` | MIMEスニッフィング防止 |
| `Referrer-Policy` | リファラー情報の漏洩制御 |
| `Permissions-Policy` | カメラ・位置情報等のブラウザAPI制限 |

---

## アンチパターン vs 正しい設計

| アンチパターン | 問題点 | 正しい設計 |
|---|---|---|
| `Allow-Origin: *` + 認証Cookie | 任意サイトからのCORS要求を許可してしまう | `*` は認証不要なパブリックAPIのみ。認証ありは具体的オリジンを指定 |
| CSP未設定 | XSS時に任意スクリプトが実行される | `default-src 'self'` から始め段階的に厳格化 |
| `unsafe-inline` を CSP に追加 | インラインスクリプトを許可してしまいXSS防御が無効化 | nonceベースのCSPに移行 |
| CORSを全エンドポイントに一括適用 | 本来公開不要なAPIも外部から叩ける | エンドポイントごとに必要なオリジンだけ許可 |
| HSTSなし | HTTPへのダウングレード攻撃が可能 | `max-age=63072000; includeSubDomains; preload` |

---

## コード/設計例（FastAPI）

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request

ALLOWED_ORIGINS = ["https://app.example.com", "https://admin.example.com"]

app = FastAPI()

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
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
        response.headers["Content-Security-Policy"] = (
            "default-src 'self'; script-src 'self'; object-src 'none'"
        )
        return response

app.add_middleware(SecurityHeadersMiddleware)
```

---

## トレードオフ

| 観点 | 厳格にする | 緩くする |
|---|---|---|
| セキュリティ | 高い | 低い |
| 開発体験 | ローカルでCORSエラーが頻発 | 快適だが本番で事故 |
| CDN・外部リソース | CSPが複雑化する | 設定シンプル |
| 段階的移行 | `report-only` モードで違反を先に観測できる | 一発で全適用は壊れやすい |

**推奨アプローチ**: `Content-Security-Policy-Report-Only` で2週間観測 → 違反ゼロを確認 → 本適用

---

## チェックリスト

- [ ] `Access-Control-Allow-Origin: *` を認証付きAPIで使っていないか確認した
- [ ] CSP を `report-only` で導入し、違反ログを収集している
- [ ] `X-Frame-Options: DENY` または CSP `frame-ancestors 'none'` でクリックジャッキングを防いでいる
- [ ] HSTS を設定し、HTTPへのダウングレードを防いでいる
- [ ] [securityheaders.com](https://securityheaders.com) でヘッダー評価をスコアA以上にしている
