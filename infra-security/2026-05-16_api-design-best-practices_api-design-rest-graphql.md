# API設計のベストプラクティス — REST vs GraphQL・バージョニング・レート制限・認証

## 概要

API設計はシステムの「境界線」を決める作業であり、一度公開すると変更コストが高い。  
誤った設計は下流チームへの依存・バージョン爆発・セキュリティ事故を引き起こす。  
AI時代においても「何をエンドポイントとして切るか・誰に何を返すか」はビジネスロジックと直結しており、設計判断力が問われる領域。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|---|---|---|
| クライアント多様性 | 低い（1クライアント想定） | 高い（モバイル・Web混在） |
| Over-fetching/Under-fetching | 起きやすい | クライアント側が解決 |
| キャッシュ | HTTP標準で容易 | POSTが多くCDNキャッシュが難しい |
| 学習コスト | 低い | スキーマ設計・リゾルバが必要 |
| FastAPIとの相性 | ◎ ネイティブサポート | △ Strawberryなど追加ライブラリ |

**判断軸**: BFF（Backend for Frontend）が不要で1種類のクライアントならREST。複数クライアントが異なる粒度でデータを要求するならGraphQL。

---

### URLとバージョニング設計

- **URL例**: `/api/v1/users/{user_id}/orders`
- バージョンはURLパス（`/v1/`）に埋め込むのが最もシンプル
- ヘッダーバージョニング（`Accept: application/vnd.app.v2+json`）はCDNキャッシュが壊れやすい
- **後方互換の保ち方**:
  - フィールド追加はOK、フィールド削除・型変更はNG
  - 削除する場合は `deprecated: true` をレスポンスに返し、90日後に廃止

### HTTPメソッドとステータスコード

- `GET` → 冪等・副作用なし → キャッシュ可
- `POST` → リソース作成 → `201 Created` + `Location` ヘッダー
- `PUT` → リソース全体置換 → `200` or `204`
- `PATCH` → 部分更新 → `200` or `204`
- `DELETE` → `204 No Content`（ボディ不要）
- エラー: `400` バリデーション / `401` 未認証 / `403` 権限なし / `404` 未発見 / `429` レート制限超過

---

## アンチパターン vs 正しい設計

### アンチパターン
- `POST /getUser` → GETをPOSTで代替（キャッシュ不可・意味不明）
- エラーを常に `200 OK` で返す（モニタリングが死ぬ）
- バージョンなしでフィールドを削除する（クライアントが突然壊れる）
- 認証トークンをクエリパラメータに載せる（ログに漏れる）

### 正しい設計
- リソース名は複数形の名詞: `/users`, `/orders`
- エラーはRFC 7807（Problem Details）形式で統一
- トークンは `Authorization: Bearer <token>` ヘッダーのみ
- ページネーションは `cursor` ベース（OFFSETは大量データで遅い）

---

## コード/設計例（FastAPI）

```python
from fastapi import APIRouter, Depends, HTTPException, Header
from typing import Optional

router = APIRouter(prefix="/api/v1")

@router.get("/users/{user_id}/orders", status_code=200)
async def list_orders(
    user_id: str,
    cursor: Optional[str] = None,
    limit: int = 20,
    authorization: str = Header(...),
):
    # Bearerトークン検証はDependsで共通化
    # cursor-based pagination
    return {
        "data": [...],
        "next_cursor": "xxx",
        "limit": limit,
    }

# エラーはRFC 7807形式
raise HTTPException(
    status_code=422,
    detail={"type": "validation_error", "title": "Invalid input", "detail": "..."}
)
```

---

## レート制限の設計

- **実装位置**: Cloudflare（エッジ）→ Cloud Run（アプリ）の2層で防御
- **識別キー**: `user_id`（認証済み）または `IP`（未認証）
- **上限例**: 認証済み = 1000req/分、未認証 = 60req/分
- **レスポンスヘッダー**: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `Retry-After`
- **ストレージ**: Redis（またはCloud Memorystore）でカウンタ管理

---

## トレードオフ

| 判断 | メリット | デメリット |
|---|---|---|
| URLバージョニング | シンプル・CDN対応 | URLが冗長になる |
| GraphQL | Over-fetch解消 | N+1問題・キャッシュ複雑 |
| cursor pagination | 大量データでも高速 | ページ番号指定ができない |
| 厳格なレート制限 | 安全 | 正規ユーザーの体験悪化リスク |

---

## チェックリスト

- [ ] URLはリソース名（名詞複数形）、動詞を含まない
- [ ] エラーレスポンスに一貫したスキーマ（RFC 7807推奨）を使っている
- [ ] 認証トークンはAuthorizationヘッダーのみ（クエリパラメータNG）
- [ ] バージョン廃止前に `deprecated` 通知期間を設けている
- [ ] レート制限の識別キーとストレージが決まっている
