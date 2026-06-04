# API設計のベストプラクティス（REST vs GraphQL・バージョニング・認証）

## 概要

API設計はシステムの「契約」であり、一度公開すると変更コストが高い。
AI時代においても「どういうインターフェースを設計するか」はエンジニアの核心的な仕事。
FastAPI + Firebase Auth + Cloud Run スタックにおいて、正しい設計が可用性・セキュリティ・開発速度を左右する。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|---|---|---|
| 向いている用途 | リソース中心、シンプルなCRUD | 複雑なデータ取得、モバイル最適化 |
| キャッシュ | HTTPキャッシュがそのまま使える | クエリごとに異なりキャッシュが難しい |
| 型安全 | OpenAPIで後付け | スキーマファーストで自然に型安全 |
| N+1問題 | ルーティングで回避しやすい | DataLoader等で明示的に解決が必要 |
| チーム習熟度 | 一般的 | 学習コスト高め |

**判断基準**: BFF（Backend for Frontend）層やモバイル向けの複雑な集計はGraphQL、それ以外はRESTが無難。

### バージョニング戦略

- **URLパスバージョニング** (`/v1/users`) — 最も明確で推奨
- **ヘッダーバージョニング** (`Accept: application/vnd.api+json;version=2`) — クリーンだがテストが面倒
- **クエリパラメータ** (`?version=2`) — 非推奨。キャッシュが効きにくい

**後方互換性の原則**:
- フィールドの追加はOK
- フィールドの削除・型変更は破壊的変更 → バージョンを上げる
- 非推奨フィールドは`deprecated`フラグ＋最低6ヶ月の猶予期間

### 認証フロー（Firebase Auth + FastAPI）

```
Client → Firebase Auth → IDトークン取得
Client → FastAPI (Authorization: Bearer <IDトークン>)
FastAPI → firebase_admin.auth.verify_id_token() → uid取得
FastAPI → DBクエリ（RLSがuidで自動フィルタ）
```

### レート制限の設計

- **エンドポイント別に設定**: 認証エンドポイントは厳しく（5req/min）、読み取りは緩く
- **ユーザー単位 + IP単位** の両方で実装
- **429 Too Many Requests** + `Retry-After` ヘッダーで返す

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| `/getUsers`, `/createUser` (動詞URL) | `/users` (名詞URL + HTTPメソッドで動詞表現) |
| エラー全部200で返す | 400/401/403/404/409/429/500 を正しく使い分ける |
| バージョン管理なし | 最初から `/v1/` プレフィックス |
| 全フィールドを返す | 必要なフィールドのみ（ページネーション必須） |
| 認証トークンをURLに含める | Authorizationヘッダーのみ使用 |
| エラーに詳細スタックトレースを返す | 外部には汎用メッセージ、ログにのみ詳細 |

---

## コード例（FastAPI での実装）

```python
from fastapi import APIRouter, Depends, HTTPException, status
from firebase_admin import auth

router = APIRouter(prefix="/v1")

async def get_current_user(token: str = Depends(oauth2_scheme)):
    try:
        decoded = auth.verify_id_token(token)
        return decoded["uid"]
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")

@router.get("/users/me")
async def get_me(uid: str = Depends(get_current_user)):
    return {"uid": uid}

@router.get("/items")
async def list_items(
    uid: str = Depends(get_current_user),
    limit: int = 20,   # デフォルト上限設定
    cursor: str = None # オフセットではなくカーソルページネーション
):
    ...
```

---

## トレードオフ

### カーソルページネーション vs オフセットページネーション

- **オフセット** (`?page=2&size=20`): 実装簡単だが大量データで遅くなる（OFFSET 10000はフルスキャン）
- **カーソル** (`?after=cursor_token`): パフォーマンス良好、リアルタイムデータに強い。ただしランダムアクセス不可

→ **推奨**: 本番は最初からカーソルベース

### 同期 vs 非同期レスポンス

- 処理が3秒以内 → 同期（202でなく200返す）
- 処理が長い → `POST /jobs` → `202 Accepted` + `Location: /jobs/{id}` でポーリング
- イベント通知が必要 → WebSocket or SSE（Cloud Runはロングポーリング非推奨）

---

## チェックリスト

- [ ] URLは名詞・複数形、HTTPメソッドで操作を表現している
- [ ] 全エンドポイントに `/v1/` プレフィックスがある
- [ ] 認証はAuthorizationヘッダーのみ（URLにトークンを含めない）
- [ ] エラーレスポンスは外部に詳細を漏らさない（スタックトレースを返さない）
- [ ] ページネーションに上限がある（無制限取得を防ぐ）
