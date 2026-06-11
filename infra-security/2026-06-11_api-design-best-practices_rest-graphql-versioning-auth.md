# API設計のベストプラクティス（REST vs GraphQL・バージョニング・レート制限・認証）

## 概要

APIはシステムの「契約」であり、一度公開すると変更コストが高い。AI時代においても、外部連携・マイクロサービス間通信・フロントエンドとの境界線はAPIで定義される。
「とりあえず動くエンドポイント」ではなく、**スケール・変更容易性・セキュリティを最初から織り込んだ設計**が求められる。
FastAPI + Cloud Run + Firebase Auth スタックでの判断基準を中心に整理する。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いているケース | CRUD中心・キャッシュ重視・公開API | 複雑な関連データ取得・モバイル最適化 |
| キャッシュ | HTTP標準（CDN利用可） | デフォルト難しい（GET+永続クエリで対応） |
| N+1問題 | ルーティング設計で回避 | DataLoader必須 |
| 型安全 | OpenAPI/スキーマ生成 | スキーマファースト（型が強い） |
| 学習コスト | 低 | 高 |

**判断基準（FastAPIスタックなら）:**
- BFF（Backend for Frontend）が必要 → GraphQL検討
- 公開API・サードパーティ連携・CDNキャッシュ重視 → REST
- **スタート時はRESTで十分。GraphQLは複雑さを持ち込む**

### バージョニング戦略

- **URLパス方式** `/v1/users` — 最もシンプル、CDNキャッシュ相性○
- **ヘッダー方式** `Accept: application/vnd.myapp.v2+json` — URLが綺麗だが実装複雑
- **クエリパラメータ** `?version=2` — 非推奨。ブックマーク・キャッシュ汚染リスク

**推奨:** URLパス方式 + セマンティックバージョン（メジャーのみ）  
変更互換性のルール:
- **後方互換（フィールド追加）** → バージョン上げ不要
- **破壊的変更（フィールド削除・型変更）** → v2 に移行、v1 は非推奨期間（最低6ヶ月）を設ける

### レート制限の設計

- **なぜ必要か:** DDoS・コスト爆発・不正スクレイピング防止
- **粒度の選択:**
  - 未認証 → IP単位（厳しく）
  - 認証済 → ユーザーID/APIキー単位（緩め）
  - Tier制 → 無料/有料でレート差別化

実装位置の優先度:
1. **Cloudflare / APIゲートウェイ層** — アプリに届く前にブロック（最も効率的）
2. **ミドルウェア層（FastAPI）** — 細かい制御が必要な場合
3. **DBレイヤー** — 最後の砦（パフォーマンスコスト高）

---

## アンチパターン vs 正しい設計

### アンチパターン
- `GET /getUsers`, `POST /deleteUser` — 動詞をURLに入れる
- エラーレスポンスが全て `{"error": "Something went wrong"}` — デバッグ不能
- 認証トークンをクエリパラメータで渡す `?token=xxx` — ログに露出
- 全エンドポイントが認証なし or 全員管理者権限

### 正しい設計
- リソース名詞 + HTTPメソッドでアクション表現: `DELETE /users/{id}`
- RFC 7807準拠のエラーレスポンス（type, title, status, detail）
- 認証はAuthorizationヘッダー: `Bearer <token>`
- Firebase Auth UID で所有者確認 + Cloud Run IAMで内部API保護

---

## コード例（FastAPI + Firebase Auth）

```python
from fastapi import APIRouter, Depends, HTTPException, Request
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
router = APIRouter(prefix="/v1")

@router.get("/users/{user_id}")
@limiter.limit("60/minute")  # レート制限
async def get_user(
    request: Request,
    user_id: str,
    current_user: dict = Depends(verify_firebase_token),  # 認証
):
    # 所有者チェック（認可）
    if current_user["uid"] != user_id and not current_user.get("admin"):
        raise HTTPException(status_code=403, detail="Forbidden")
    return await fetch_user(user_id)

# RFC 7807 エラーレスポンス
def api_error(status: int, title: str, detail: str):
    raise HTTPException(status_code=status, detail={
        "type": f"https://api.example.com/errors/{title.lower()}",
        "title": title, "status": status, "detail": detail,
    })
```

---

## トレードオフ

| 設計判断 | メリット | デメリット |
|---------|---------|-----------|
| URLバージョニング | 明示的・CDN親和性 | URL冗長・複数バージョン維持コスト |
| 厳格なレート制限 | 安全・コスト予測可 | 正規ユーザーをブロックリスク |
| GraphQL採用 | フロント柔軟性高 | キャッシュ複雑・N+1・権限制御が難 |
| 細粒度の認可チェック | セキュリティ堅牢 | 実装コスト・パフォーマンス低下 |

**FastAPI + Cloud Run 推奨構成:**
- REST + URLバージョニング
- Cloudflareでレート制限（IP単位）+ slowapiでユーザー単位
- Firebase Auth（認証）+ RLS（認可）の二層防御

---

## チェックリスト

- [ ] エラーレスポンスに type/status/detail が含まれているか
- [ ] 認証トークンはヘッダーのみで渡しているか（URLに含めない）
- [ ] レート制限はアプリ層だけでなく、CDN/ゲートウェイ層でも設定しているか
- [ ] 破壊的変更時に旧バージョンの非推奨期間を設けているか
- [ ] 全エンドポイントで認証・認可（所有者確認）が適切に実装されているか
