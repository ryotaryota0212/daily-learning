# API設計のベストプラクティス：REST vs GraphQL・バージョニング・レート制限・認証

## 概要

API設計はシステムの「契約」であり、一度公開すると変更コストが高い。
FastAPI + Firebase Auth スタックでは、設計の初期判断（RESTかGraphQLか、バージョニング方式、認証方式）が後の保守コストを大きく左右する。
「とりあえず動くAPI」ではなく、スケーラブルで変更に強い設計を最初から選ぶ力が問われる。

---

## 仕組みの要点

### REST vs GraphQL の判断基準

| 観点 | REST | GraphQL |
|---|---|---|
| 向いているケース | CRUD中心、外部公開API | 複雑なデータ取得、BFF層 |
| オーバーフェッチ | 発生しやすい | 必要フィールドのみ取得 |
| キャッシュ | HTTP標準が使える | クエリ単位でキャッシュ難 |
| スキーマ | 各エンドポイントで定義 | 一元的な型定義 |
| 学習コスト | 低い | 高い |

**判断ルール：**
- 外部パートナー向けAPI・公開API → REST
- モバイル/複数クライアントで異なるデータ取得が必要 → GraphQL検討
- チームがGraphQLに不慣れなら → REST + 複数エンドポイントで十分

### バージョニング戦略

- **URLパス方式**（推奨）: `/api/v1/users` → 明示的、キャッシュしやすい
- **ヘッダー方式**: `Accept: application/vnd.api+json;version=1` → URLが汚れないが発見しにくい
- **廃止ポリシー**: 古いバージョンは最低6ヶ月サポート、Sunsetヘッダーで通知

### 認証設計（Firebase Auth + FastAPI）

- Firebase IDトークンをBearerトークンとして送信
- エンドポイントでトークン検証→UIDをリクエストコンテキストに載せる
- サービス間通信はGoogle-managed Service Accountで認証（IDトークンと別扱い）

### レート制限の設計

- **単位**: ユーザーUID単位 > IPアドレス単位（認証済みAPIならUID単位が正確）
- **アルゴリズム**: Token Bucket（バースト許容）が実用的
- **返却**: `429 Too Many Requests` + `Retry-After` ヘッダー

---

## アンチパターン vs 正しい設計

| アンチパターン | 問題 | 正しい設計 |
|---|---|---|
| `/getUser`, `/deleteUser` のような動詞URL | RESTの原則違反、一貫性がない | `GET /users/{id}`, `DELETE /users/{id}` |
| 全エラーを200で返す | クライアントが状態判断できない | 適切なHTTPステータスコード使用 |
| バージョニングなし | 後から変更できなくなる | 最初から `/api/v1/` を使う |
| 認証なしでデバッグエンドポイント公開 | 情報漏洩リスク | 全エンドポイントに認証必須 |
| エラーレスポンスにスタックトレース | 内部実装が露出 | 汎用エラーメッセージのみ返す |

---

## コード/設計例（FastAPI）

```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import firebase_admin.auth as fa

app = FastAPI()
security = HTTPBearer()

async def get_current_user(creds: HTTPAuthorizationCredentials = Depends(security)):
    try:
        decoded = fa.verify_id_token(creds.credentials)
        return decoded["uid"]
    except Exception:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED)

@app.get("/api/v1/users/me")
async def get_me(uid: str = Depends(get_current_user)):
    return {"uid": uid}

# レート制限はミドルウェアで実装（slowapi等）
# from slowapi import Limiter
# @limiter.limit("60/minute")
```

---

## トレードオフ

- **REST の細かいエンドポイント vs GraphQL の柔軟取得**：REST は個別キャッシュが効くが、クライアント要求の変化に弱い。GraphQL は柔軟だが N+1問題・認可の複雑さが増す
- **URLバージョニング vs ヘッダーバージョニング**：URLは明示的でツールサポートが良いが、URL設計が冗長になる
- **UID単位レート制限 vs IP単位**：UIDは正確だが認証後のみ有効。未認証エンドポイントはIPに頼らざるを得ない
- **厳密なバリデーション vs 寛容なパース**：厳密にすると後方互換が壊れやすい。ただし緩すぎると不正データが通る

---

## チェックリスト

- [ ] 全エンドポイントに適切なHTTPメソッドとステータスコードを使っているか
- [ ] URLに `/api/v1/` のバージョンプレフィックスを含めているか
- [ ] 認証済みエンドポイントはFirebase IDトークン検証を必ず通るか
- [ ] エラーレスポンスに内部情報（スタックトレース、DB名）が含まれていないか
- [ ] レート制限（429）と `Retry-After` ヘッダーを実装しているか
