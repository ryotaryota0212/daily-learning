# API設計のベストプラクティス（REST・バージョニング・レート制限・認証）

## 概要

APIはシステムの「契約」であり、一度公開すると変更コストが高い。「動くAPIを素早く作る」より「壊れにくく、使いやすく、安全なAPIを最初に設計する」方が長期コストが圧倒的に低い。FastAPI + Cloud Run + Firebase Auth スタックでは特に認証・バージョニング・レート制限の設計判断が重要。

---

## 仕組みの要点

### RESTの原則（省略しがちな部分）
- **ステートレス**: サーバーはリクエスト間の状態を保持しない。セッションはクライアント（JWTなど）が持つ
- **リソース指向**: `/getUser` はNG。`GET /users/{id}` が正しい（動詞ではなく名詞）
- **HTTPメソッドの正しい使い方**:
  - `GET` → 副作用なし、冪等
  - `POST` → 作成、非冪等
  - `PUT` → 完全置換、冪等
  - `PATCH` → 部分更新
  - `DELETE` → 削除、冪等

### ステータスコードの正しい使い方
| コード | 使い場面 |
|--------|----------|
| 200 | 成功（GET, PATCH） |
| 201 | リソース作成成功（POST） |
| 204 | 成功・レスポンスボディなし（DELETE） |
| 400 | クライアントの入力ミス |
| 401 | 未認証（トークンなし・無効） |
| 403 | 認証済みだが権限なし |
| 404 | リソース不存在 |
| 409 | 競合（重複登録など） |
| 422 | バリデーションエラー（FastAPIデフォルト） |
| 429 | レート制限超過 |
| 500 | サーバー内部エラー |

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|----------------|------------|
| `/api/getUsers` | `GET /api/v1/users` |
| エラーでも `200 OK` を返す | 適切な4xx/5xx を返す |
| バージョン管理なし | URLパス `/v1/` でバージョン管理 |
| エラーレスポンスが不統一 | 統一エラーフォーマット |
| 認証チェックを各エンドポイントに実装 | FastAPIのDependency Injectionで一元管理 |
| 全フィールドを常に返す | 必要なフィールドのみ（レスポンス型を明示） |

---

## コード例（FastAPI + Firebase Auth）

```python
# 統一エラーレスポンスとDI認証の実装例
from fastapi import FastAPI, Depends, HTTPException, Header
from firebase_admin import auth

app = FastAPI()

async def verify_token(authorization: str = Header(...)):
    token = authorization.removeprefix("Bearer ")
    try:
        return auth.verify_id_token(token)
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")

# v1 ルーターでバージョン管理
from fastapi import APIRouter
router_v1 = APIRouter(prefix="/api/v1")

@router_v1.get("/users/{user_id}")
async def get_user(user_id: str, claims=Depends(verify_token)):
    if claims["uid"] != user_id:
        raise HTTPException(status_code=403, detail="Forbidden")
    return {"id": user_id, "name": "..."}
```

---

## バージョニング戦略

- **URLパス方式**（推奨）: `/api/v1/`, `/api/v2/` → 最も明示的
- **ヘッダー方式**: `API-Version: 2` → URLが汚れないがデバッグしにくい
- **クエリパラメータ方式**: `?version=1` → 非推奨（キャッシュに影響）

**バージョンアップの判断基準**:
- レスポンスの型が変わる → メジャーバージョンアップ
- フィールド追加のみ → 後方互換、バージョンアップ不要
- 古いバージョンは最低6ヶ月は維持し、廃止予告を `Deprecation` ヘッダーで通知

---

## レート制限設計

- **Cloud Run + Cloudflare WAF** でIPベースのレート制限をエッジで実装
- アプリレベルでは Redis でスライディングウィンドウ方式が堅牢
- レスポンスヘッダーで状態を通知:
  ```
  X-RateLimit-Limit: 100
  X-RateLimit-Remaining: 42
  X-RateLimit-Reset: 1724800000
  ```
- レート制限超過時は `429 Too Many Requests` + `Retry-After` ヘッダーを返す

---

## トレードオフ

| 観点 | REST | GraphQL |
|------|------|---------|
| シンプルさ | ○ 直感的 | △ スキーマ学習コスト |
| 過剰取得防止 | △ 設計次第 | ○ クライアントが指定 |
| キャッシュしやすさ | ○ GETがキャッシュ可能 | △ POSTが基本でキャッシュ困難 |
| 採用場面 | 汎用API・モバイル | BFF・複雑なデータ取得 |

**現実的判断**: FastAPI + Cloud Run なら REST 一択。GraphQL は専用クライアントと BFF がある場合のみ検討。

---

## チェックリスト

- [ ] URLはリソース指向（動詞なし）で設計されているか
- [ ] HTTPステータスコードを正しく使い分けているか
- [ ] バージョン管理（`/v1/`）を最初から組み込んでいるか
- [ ] 認証はDependency Injectionで一元管理されているか
- [ ] レート制限と `429` レスポンスが実装されているか
