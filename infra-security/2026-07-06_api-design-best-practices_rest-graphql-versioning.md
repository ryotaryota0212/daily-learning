# API設計のベストプラクティス — REST vs GraphQL・バージョニング・認証

## 概要

APIはシステムの「契約」であり、一度公開すると変更コストが極めて高い。
AI時代においても「どういうAPI設計にすればクライアントとサーバーを独立して進化させられるか」を判断できることが設計力の核心。
FastAPI + Firebase Auth + Cloud Run スタックを前提に、壊れにくいAPI設計の原則を整理する。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いているケース | リソース操作が明確、外部公開API | 画面ごとに必要データが異なる、BFF層 |
| オーバーフェッチ | 起きやすい | 起きにくい |
| キャッシュ | HTTP標準で容易 | クエリハッシュが必要 |
| 学習コスト | 低 | 高 |
| **結論** | スモールチームの初期は REST を選ぶ | 画面数が多くフロントが複数ある場合に検討 |

### バージョニング戦略

- **URLパス方式** `/v1/users` — 最も一般的。キャッシュしやすく明示的
- **ヘッダー方式** `Accept: application/vnd.api+json;version=2` — URLが汚れないが複雑
- **クエリパラメータ** `/users?version=2` — 推奨しない（キャッシュが効きにくい）

推奨: **URLパス方式**を採用し、`/v1/` を廃止するまで最低6ヶ月の移行期間を設ける。

### 認証フロー（Firebase Auth + FastAPI）

1. クライアントがFirebase AuthでIDトークン取得
2. `Authorization: Bearer <id_token>` でAPIリクエスト
3. FastAPI側でFirebase Admin SDKによるトークン検証
4. 検証済みの `uid` をDBクエリの条件に使う

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|-------------|------------|
| `POST /getUser` — 動詞をパスに入れる | `GET /users/{id}` — HTTPメソッドで操作を表す |
| レスポンスに常に `200 OK` を返す | `404 / 422 / 401` など適切なステータスコードを使う |
| エラーメッセージにスタックトレースを含める | クライアント向けには汎用メッセージ、サーバーログに詳細を残す |
| 全フィールドを1エンドポイントで返す | ユースケースに応じた粒度でエンドポイントを分ける |
| バージョンなしでAPIを公開 | 最初から `/v1/` プレフィックスを付ける |

---

## コード例（FastAPI + Firebase Auth）

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer
from firebase_admin import auth

bearer = HTTPBearer()

async def get_current_user(token=Depends(bearer)):
    try:
        decoded = auth.verify_id_token(token.credentials)
        return decoded  # {"uid": "...", "email": "..."}
    except Exception:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED)

@app.get("/v1/users/me")
async def get_me(user=Depends(get_current_user)):
    return {"uid": user["uid"]}
```

---

## トレードオフ

- **URLパスバージョニング**: 明示的でわかりやすいが、v2移行後にv1の保守コストが残る
- **GraphQL**: オーバーフェッチを防げるが、N+1問題・認可の複雑化・キャッシュ戦略が難しくなる
- **細かいエンドポイント分割**: 明確で変更しやすいが、エンドポイント数が増えクライアント側の呼び出し数が増える
- **スキーマバリデーション（Pydantic）**: 安全性が上がるが厳しすぎると後方互換が壊れやすい

---

## チェックリスト

- [ ] 最初から `/v1/` プレフィックスをつけているか
- [ ] HTTPステータスコードを正しく使っているか（200/201/400/401/403/404/422/500）
- [ ] エラーレスポンスにスタックトレースが含まれていないか
- [ ] 認証はFirebase IDトークンをサーバー側で検証しているか（クライアント信頼しない）
- [ ] レート制限・タイムアウトをCloud Run / Cloudflareレベルで設定しているか
