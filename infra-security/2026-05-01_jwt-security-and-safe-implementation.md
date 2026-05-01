# JWT の仕組み・脆弱性・安全な実装方法

## 概要

JWT (JSON Web Token) は API 認証で広く使われるトークン形式。Firebase Auth も内部で JWT (ID Token) を発行する。
署名により改ざんを検知できるが「暗号化」ではないため payload は誰でも読める。歴史的に `alg: none` 受理や鍵取り違えなど致命的脆弱性が多く、自前検証では事故が起きやすい。
「動くこと」より「壊れた時に何が起きるか」を意識した検証実装が肝。

## 仕組みの要点

- 構造: `Header.Payload.Signature` の3つを `.` で連結し base64url エンコード
- Header: `alg`(署名アルゴリズム), `kid`(鍵ID)
- Payload: `iss`(発行者), `aud`(受信者), `exp`(失効), `iat`(発行時刻), `sub`(主体)などのクレーム
- 署名方式は対称鍵 HS256 と非対称鍵 RS256/ES256
- Firebase ID Token は RS256 で署名、公開鍵は Google の JWKS エンドポイントで配布
- 検証: 署名 + `iss`/`aud`/`exp` を必ず確認

## 主な脆弱性 vs 正しい設計

| アンチパターン | 何が起きるか | 正しい設計 |
| --- | --- | --- |
| `alg: none` を受理 | 署名なしで通る | 期待する alg をホワイトリスト固定 |
| alg confusion (RS256→HS256) | 公開鍵を HMAC キーに使われ偽造 | 鍵と alg を厳格に紐付ける |
| `exp` を見ない | 永久に有効なトークン | 必ず期限検証、リーウェイは最小 |
| `aud`/`iss` 未検証 | 別サービスのトークンが通る | 自サービスの値で完全一致 |
| LocalStorage に保存 | XSS で盗まれる | HttpOnly + Secure + SameSite Cookie |
| 長すぎる有効期間 | 漏洩時の被害大 | Access は短命(15分)、Refresh で更新 |
| 自前で署名検証を実装 | バグ混入リスク | 実績あるライブラリ (PyJWT, firebase-admin) |

## コード例: FastAPI で Firebase ID Token 検証

```python
from fastapi import Depends, HTTPException, Request
from firebase_admin import auth, initialize_app

initialize_app()

async def verify_token(request: Request) -> dict:
    header = request.headers.get("Authorization", "")
    if not header.startswith("Bearer "):
        raise HTTPException(401, "missing bearer")
    token = header.removeprefix("Bearer ")
    try:
        # 署名・exp・iss・aud・revoked をまとめて検証
        decoded = auth.verify_id_token(token, check_revoked=True)
    except auth.RevokedIdTokenError:
        raise HTTPException(401, "revoked")
    except auth.ExpiredIdTokenError:
        raise HTTPException(401, "expired")
    except Exception:
        raise HTTPException(401, "invalid token")
    return decoded  # uid, email などを含む

@app.get("/me")
def me(user=Depends(verify_token)):
    return {"uid": user["uid"]}
```

ポイント: `verify_id_token` は JWKS キャッシュ・alg 固定・clock skew 許容を内包しており、自前実装より安全。

## トレードオフ

| 観点 | JWT (ステートレス) | セッション (ステートフル) |
| --- | --- | --- |
| スケール | DB 不要で水平スケール容易 | セッションストア必須 |
| 無効化 | 即時失効が困難 | サーバ側で削除すれば即時 |
| サイズ | Cookie/ヘッダが大きい | セッションIDのみ |
| 複数サービス連携 | 公開鍵検証で容易 | 共有ストア必要 |
| 漏洩時のリスク | exp まで使われる | 即無効化可 |

短命 Access Token + Refresh Token + サーバ側失効リスト (またはトークンバージョン) の併用が現実的な落とし所。

## 設計時のチェックリスト

- [ ] `alg` をホワイトリスト固定し `none` を拒否しているか
- [ ] `iss` / `aud` / `exp` をすべて検証しているか
- [ ] Access Token は短命 (15分以内)、Refresh は別経路で管理しているか
- [ ] ブラウザ保存は HttpOnly + Secure + SameSite=Lax/Strict Cookie か
- [ ] 鍵ローテーション (JWKS の `kid`) と失効手段 (revoke / token version) を持っているか
