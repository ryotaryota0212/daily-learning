# API設計のベストプラクティス：REST vs GraphQL・バージョニング・レート制限・認証

## 概要

API設計の良し悪しはシステムの長期的な保守性・拡張性・セキュリティに直結する。特にAI時代は「とりあえず動くAPI」でなく「クライアントとの契約として機能するAPI」が求められる。設計ミスは後工程で莫大なコストになるため、初期設計での判断力が重要。FastAPI + Firebase Auth + Cloud Run 構成での実践ポイントを整理する。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 学習コスト | 低い | 高い |
| Over-fetching防止 | 難しい | 得意 |
| キャッシュ | CDN・HTTPキャッシュがそのまま効く | クエリ単位でカスタム実装が必要 |
| スキーマ変更 | バージョニング必要 | 後方互換しやすい |
| 向いている用途 | シンプルなCRUD、公開API | 複雑なデータグラフ、モバイル最適化 |

**結論**：FastAPI + Cloud Run 構成では REST が基本。複数クライアント（Web/Mobile）で取得フィールドが大幅に異なる場合のみ GraphQL を検討。

### バージョニング戦略

- **URLパス方式**（推奨）：`/api/v1/users` → 明示的でキャッシュしやすい
- **ヘッダー方式**：`Accept: application/vnd.api+json;version=1` → URL汚染なし、デバッグ難
- **クエリパラメータ**：`/users?version=1` → 非推奨、キャッシュ制御が困難

**バージョンアップの判断基準**：
- レスポンスフィールドの削除・型変更 → v2 必須
- フィールド追加・任意パラメータ追加 → 後方互換、v2 不要
- 廃止予定は `Deprecation` / `Sunset` ヘッダーで事前通知

### レート制限の設計

- **ユーザー単位**（Firebase UID）：`100 req/min/user` 等
- **IP単位**：未認証エンドポイント保護
- **エンドポイント単位**：重い処理（AI生成等）は個別に厳しく
- レスポンスヘッダーで残量を返す：`X-RateLimit-Remaining`, `X-RateLimit-Reset`
- 429 Too Many Requests + `Retry-After` ヘッダーをセット

### 認証設計

- 全エンドポイントは原則認証必須（Firebase ID Token を Bearer トークンで送信）
- `/health`, `/metrics` 等の内部エンドポイントは IP制限 or Cloud Run internal
- サービス間通信（Cloud Run to Cloud Run）は IAM サービスアカウント + OIDC トークン

---

## アンチパターン vs 正しい設計

| アンチパターン | 問題 | 正しい設計 |
|---|---|---|
| `/getUser`, `/createUser` 等の動詞URL | RESTの原則違反、意図が予測不能 | `/users/{id}` + HTTPメソッドで区別 |
| 200 OK でエラーを返す | クライアントのハンドリングが破綻 | 適切なHTTPステータスコード（400/401/403/404/422/500） |
| バージョニングなしで破壊的変更 | クライアント側で即時障害 | `Deprecation` ヘッダーで移行期間を設ける |
| レート制限なし | DDoS・スクレイピングに無防備 | 必ずレート制限を設計に含める |
| エラーレスポンスにスタックトレース | 内部構造の露出 | 汎用メッセージ + エラーコードのみ返す |

---

## コード例（FastAPI + Firebase Auth + レート制限）

```python
from fastapi import FastAPI, Depends, HTTPException, Request
from slowapi import Limiter
from slowapi.util import get_remote_address
import firebase_admin
from firebase_admin import auth

limiter = Limiter(key_func=get_remote_address)
app = FastAPI()

async def verify_firebase_token(request: Request) -> dict:
    token = request.headers.get("Authorization", "").removeprefix("Bearer ")
    try:
        return auth.verify_id_token(token)
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")

@app.get("/api/v1/users/{user_id}")
@limiter.limit("60/minute")
async def get_user(
    request: Request,
    user_id: str,
    claims: dict = Depends(verify_firebase_token)
):
    if claims["uid"] != user_id:
        raise HTTPException(status_code=403, detail="Forbidden")
    return {"id": user_id, "email": claims.get("email")}
```

---

## トレードオフ

| 選択 | メリット | デメリット |
|------|----------|------------|
| URLバージョニング | 明確・CDN対応 | URL設計が冗長になる |
| GraphQL | フィールド選択・後方互換 | キャッシュ複雑・学習コスト |
| ユーザー単位レート制限 | 公平性が高い | 認証前のエンドポイントに使えない |
| 厳しいレート制限 | 安全 | 正常ユーザー体験を損なうリスク |

---

## チェックリスト

- [ ] 破壊的変更前に `/api/v1` → `/api/v2` のバージョニング計画があるか
- [ ] 全エンドポイントに Firebase ID Token 検証が適用されているか
- [ ] レート制限（`slowapi` 等）が認証前・後の両方に設定されているか
- [ ] エラーレスポンスに内部情報（スタックトレース・DB構造）が含まれていないか
- [ ] `Deprecation` + `Sunset` ヘッダーで廃止予定 API をクライアントに通知しているか
