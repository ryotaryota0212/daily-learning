# OAuth 2.0 / OpenID Connect の仕組みと脅威モデル

## 概要

OAuth 2.0は**認可**（何にアクセスできるか）のプロトコル、OIDCは**認証**（誰であるか）を追加した拡張仕様。
Firebase Authを使っていても内部でOIDCが動いており、仕組みを知らないと設定ミスや脆弱性に気づけない。
「トークンをどう発行・検証するか」の設計判断がシステム全体のセキュリティを左右する。

---

## 仕組みの要点

### OAuth 2.0 の登場人物

| 役割 | 説明 | 例 |
|------|------|-----|
| Resource Owner | リソースの持ち主（ユーザー） | ログインするユーザー |
| Client | アクセスしたいアプリ | FastAPIバックエンド |
| Authorization Server | トークンを発行するサーバー | Firebase Auth / Google |
| Resource Server | 保護されたAPIを提供 | 自分のCloud Run API |

### 認可コードフロー（最も安全）

```
Browser → /authorize?response_type=code&state=xyz → Auth Server
Auth Server → redirect_uri?code=ABC&state=xyz → Browser
Browser → POST /token {code=ABC, verifier=...} → Auth Server
Auth Server → {access_token, id_token, refresh_token} → Browser
Browser → GET /api/data {Authorization: Bearer access_token} → Resource Server
```

### OIDCが追加するもの

- `id_token`（JWT）：ユーザーの身元情報（sub, email, aud, iat, exp）
- `/.well-known/openid-configuration`：エンドポイントとJWK公開鍵の自動探索
- バックエンドはJWKS URIから公開鍵を取得してid_tokenを署名検証する

### トークンの種類と特性

| トークン | 寿命 | 用途 | 漏洩リスク |
|---------|------|------|-----------|
| access_token | 短い（1h以内） | APIアクセス | 高（毎回送信） |
| id_token | 短い | 認証情報確認 | 中 |
| refresh_token | 長い（数日〜） | access_token再取得 | 最高（保管注意） |
| authorization_code | 極短（数秒〜数分） | token取得に1度だけ使用 | 中 |

---

## アンチパターン vs 正しい設計

### 認証フロー

| アンチパターン | 正しい設計 |
|--------------|-----------|
| Implicit Flow（`response_type=token`）を使う | 必ず認可コードフロー + PKCE |
| stateパラメータを省略する | stateでCSRF防止、必須 |
| redirect_uriを検証しない | 完全一致で事前登録済みURIのみ許可 |
| access_tokenをLocalStorageに保存 | HttpOnly CookieまたはメモリのみBFFパターン |

### トークン検証

| アンチパターン | 正しい設計 |
|--------------|-----------|
| `alg: none`を許可する | 許可アルゴリズムをRS256等に固定 |
| 有効期限（exp）を検証しない | exp / iat / aud / iss すべて検証 |
| JWKSを起動時だけ取得 | キャッシュしつつ定期的にローテーション追従 |

---

## コード例：FastAPIでFirebase JWTを検証

```python
from fastapi import Depends, HTTPException
from fastapi.security import HTTPBearer
import httpx, jwt, time

FIREBASE_PROJECT = "my-project"
JWKS_URI = f"https://www.googleapis.com/service_accounts/v1/jwk/securetoken@system.gserviceaccount.com"
_jwks_cache: dict = {}

async def get_jwks() -> dict:
    if _jwks_cache.get("expires", 0) > time.time():
        return _jwks_cache["keys"]
    async with httpx.AsyncClient() as c:
        r = await c.get(JWKS_URI)
    keys = {k["kid"]: k for k in r.json()["keys"]}
    _jwks_cache.update({"keys": keys, "expires": time.time() + 3600})
    return keys

async def verify_token(token: str = Depends(HTTPBearer())) -> dict:
    jwks = await get_jwks()
    header = jwt.get_unverified_header(token.credentials)
    key = jwks.get(header["kid"])
    if not key:
        raise HTTPException(401, "Unknown key")
    return jwt.decode(
        token.credentials, jwt.algorithms.RSAAlgorithm.from_jwk(key),
        algorithms=["RS256"], audience=FIREBASE_PROJECT,
        options={"require": ["exp", "iat", "sub"]}
    )
```

---

## 脅威モデル

| 脅威 | 攻撃手法 | 対策 |
|------|---------|------|
| CSRF | stateなしでコールバックを偽装 | stateをセッションと紐付け検証 |
| 認可コード横取り | redirect_uri操作 | 完全一致登録＋PKCE |
| トークン盗取 | XSSでLocalStorage読み取り | HttpOnly Cookie / BFF |
| アルゴリズム混乱 | `alg:none`や対称鍵に切替 | サーバー側でアルゴリズム固定 |
| 期限切れトークン再利用 | replay攻撃 | exp検証＋短い有効期限 |
| refresh_token漏洩 | DB/ログからの流出 | Rotation有効化、保管先を暗号化 |

---

## Firebase Auth スタック固有の注意点

- `id_token`の有効期限は**1時間固定**。`refresh_token`でのローテーションが必要
- Cloud Run → Firebase Admin SDK：サービスアカウントに `firebaseauth.viewer` 最小権限
- Emulator使用時は`FIREBASE_AUTH_EMULATOR_HOST`を設定しないと本番検証が走る
- `custom claims`でロールを付与する場合、クレーム名は予約語（`iss`等）を避ける

---

## トレードオフ

| 選択 | メリット | デメリット |
|------|---------|-----------|
| Firebase Auth（SaaS） | 実装ゼロ、MFA・OAuthプロバイダー統合済 | カスタマイズ制限、ベンダーロック |
| 自前OIDC（Keycloak等） | 完全制御、オンプレ対応 | 運用負荷大、設定ミスリスク |
| BFFパターン | フロントにトークン露出しない | バックエンド複雑化 |
| access_token短命化 | 漏洩時の被害最小 | refresh頻度増、レートリミット注意 |

---

## チェックリスト

- [ ] 認可コードフロー + PKCE を使用している（Implicit Flow は不使用）
- [ ] stateパラメータでCSRF対策、redirect_uriは事前登録済み完全一致
- [ ] id_token検証で `exp / iss / aud / alg` すべて確認している
- [ ] JWKSは定期ローテーション対応のキャッシュ戦略になっている
- [ ] refresh_tokenはHttpOnly Cookieまたは暗号化ストレージに保管している
