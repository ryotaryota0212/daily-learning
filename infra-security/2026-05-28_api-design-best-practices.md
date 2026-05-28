# API設計のベストプラクティス

## 概要

APIはシステムの「契約書」であり、一度公開すると変更コストが高い。FastAPIでエンドポイントを並べるだけでなく、「どのクライアントが何を必要としているか」「どうバージョニングするか」「誰に何を許可するか」を先に決めてから実装する姿勢がAI時代のエンジニアに求められる。設計の良し悪しがスケール・保守性・セキュリティのすべてに影響する。

---

## 仕組みの要点

### REST vs GraphQL の選び方

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いているケース | リソースが明確、キャッシュ重要 | フロントが多様、N+1問題を避けたい |
| 過剰フェッチ | 起きやすい | クライアントが必要フィールドだけ取得 |
| キャッシュ | HTTP標準（CDN対応） | クエリ複雑でCDN困難 |
| 学習コスト | 低い | スキーマ設計・型システムが必要 |
| Cloud Run + FastAPI | 相性◎（デフォルト推奨） | Strawberryで実装可能 |

**判断基準：** クライアントが1種類 or チームが小さければRESTで十分。フロントエンドの多様性・モバイル最適化が必要になってからGraphQLを検討する。

### REST設計の原則

- **リソース指向**：動詞ではなく名詞でURL設計（`GET /users/{id}` ○、`GET /getUser` ×）
- **HTTPメソッドの意味を守る**：GET=読み取り専用・冪等、POST=新規作成、PUT=全体更新・冪等、PATCH=部分更新、DELETE=削除
- **ステータスコードを正確に**：200/201/204/400/401/403/404/422/429/500
- **ページネーション**：カーソル方式（`after=<id>`）をデフォルトに。オフセット方式はデータ量増加で遅くなる
- **エラーレスポンス**：RFC 9457（Problem Details）形式に統一

### バージョニング戦略

```
# URLパス方式（推奨：明示的でキャッシュしやすい）
GET /v1/users/{id}
GET /v2/users/{id}

# ヘッダー方式（URLが変わらないが、CDNキャッシュに注意）
Accept: application/vnd.myapp.v2+json
```

- **後方互換性を保てる変更**：フィールド追加、任意パラメータ追加 → バージョン不要
- **破壊的変更**：フィールド削除・型変更・必須化 → バージョンを上げる
- **廃止ポリシー**：旧バージョンは最低6ヶ月サポート、`Deprecation`ヘッダーで告知

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| `POST /createUser`（動詞URL） | `POST /users` |
| 全エラーに200を返す | 422 Unprocessable Entity など適切なコード |
| レスポンスに毎回全フィールド | 必要フィールドのみ、または`fields`パラメータ対応 |
| バージョニングなしで破壊的変更 | URLバージョニング + 廃止スケジュールの明示 |
| 認証とレート制限なしの公開API | Firebase Auth + Cloud Run レート制限 |
| エラー詳細を本番で全公開 | 本番はエラーコードのみ、詳細はログへ |

---

## コード例（FastAPI）

```python
from fastapi import APIRouter, Depends, HTTPException, Request
from fastapi.responses import JSONResponse

router = APIRouter(prefix="/v1")

# RFC 9457 形式のエラーレスポンス
def problem_detail(status: int, title: str, detail: str):
    return JSONResponse(
        status_code=status,
        content={"type": f"https://api.example.com/errors/{status}",
                 "title": title, "detail": detail, "status": status},
        headers={"Content-Type": "application/problem+json"}
    )

@router.get("/users/{user_id}")
async def get_user(user_id: str, current_user=Depends(verify_firebase_token)):
    if current_user["uid"] != user_id:
        raise HTTPException(403)
    user = await db.fetch_user(user_id)
    if not user:
        return problem_detail(404, "User Not Found", f"user_id={user_id} は存在しません")
    return user
```

---

## レート制限・認証の設計

- **認証**：Firebase Auth ID Token を `Authorization: Bearer <token>` で受け取り、FastAPI依存性注入で検証
- **レート制限**：Cloud Run の前段に Cloud Armor か Cloudflare を置く。アプリ層では `slowapi`（Redis バックエンド）
- **スコープ制御**：Firebase Custom Claims に role を付与し、エンドポイントごとに権限チェック

```python
# レート制限（slowapi）
from slowapi import Limiter
limiter = Limiter(key_func=lambda req: req.state.user_id)

@router.post("/items")
@limiter.limit("10/minute")
async def create_item(request: Request, ...):
    ...
```

---

## トレードオフ

| 選択 | メリット | デメリット |
|------|---------|-----------|
| URLバージョニング | 明示的、CDN対応 | URLが増える |
| カーソルページネーション | 一貫性・高速 | ページ番号指定不可 |
| GraphQL | 柔軟なクエリ | N+1対策・認可設計が複雑 |
| 厳格なエラーコード | デバッグしやすい | 実装コスト増 |

---

## チェックリスト

- [ ] URLはリソース名詞、HTTPメソッドは意味通りに使っているか
- [ ] 破壊的変更はバージョンを上げているか（廃止スケジュールを通知済みか）
- [ ] エラーレスポンスが統一フォーマット（RFC 9457）になっているか
- [ ] 全エンドポイントにFirebase Auth検証が適用されているか
- [ ] レート制限がユーザー単位で設定されているか
