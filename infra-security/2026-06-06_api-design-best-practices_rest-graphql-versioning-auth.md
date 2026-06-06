# API設計のベストプラクティス：REST vs GraphQL、バージョニング、認証

## 概要

APIはシステムの「契約書」であり、一度公開したら簡単に変更できない。
設計の質がシステムの拡張性・保守性・セキュリティを長期にわたって左右する。
特に以下3点が重要：① REST vs GraphQL の適切な選択、② 破壊的変更を防ぐバージョニング、③ 全エンドポイントへの一貫した認証設計。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

- **REST**
  - リソース単位でURLを設計。操作はHTTPメソッドで表現
  - CDNキャッシュが効く（GETはURLでキャッシュキーが確定）
  - 公開API・外部向け・ドキュメント重視のケースに向いている

- **GraphQL**
  - クライアントが欲しいフィールドを指定して取得 → Over-fetching/Under-fetchingを解消
  - N+1問題が発生しやすい → DataLoaderによるバッチ処理が必須
  - 内部API・フロント要件が複雑なケースに向いている

### バージョニング戦略

- **URLパス方式**（推奨）: `/api/v1/users` → ブラウザ・CDN・curlでのテストが簡単
- **ヘッダー方式**: `Accept: application/vnd.myapi.v1+json` → URLをクリーンに保てるが複雑
- 破壊的変更（フィールド削除・型変更）は必ずバージョンアップで対応
- 旧バージョンは最低6ヶ月のサポート期間を設けてから廃止

### 認証・認可パターン

- **Bearer Token（JWT/Firebase IDトークン）**: ステートレスでスケールしやすい → 推奨
- **APIキー**: サービス間の機械的な通信向け。ローテーション運用が必要
- **OAuth2**: サードパーティ連携時に採用。スコープで権限を細かく制御
- **内部APIも認証必須**: サービス間通信でも「認証なし」は避ける

### レート制限の設計

- ユーザー単位 + IPアドレス単位の二重制限
- レスポンスヘッダーでリミット状態を通知: `X-RateLimit-Limit`, `X-RateLimit-Remaining`
- 429 Too Many Requests に `Retry-After` ヘッダーをセット

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| 動詞をURLに含む `/getUser`, `/deleteItem` | `GET /users/{id}`, `DELETE /items/{id}` |
| エラーレスポンスが毎回異なる形式 | RFC 7807 Problem Details に統一 |
| バージョン管理なしで破壊的変更 | `/v2/` に移行し旧バージョンを並行サポート |
| 一つのエンドポイントが何でも担当 | リソース・責務ごとにエンドポイントを分割 |
| 内部サービス間のAPIは認証なし | Cloud Run サービス間もIDトークンで認証 |

---

## コード例（FastAPI + Firebase Auth + Cloud Run）

```python
from fastapi import APIRouter, Depends
from fastapi.responses import JSONResponse

router = APIRouter(prefix="/api/v1")

# RFC 7807準拠の統一エラーレスポンス
def problem_detail(status: int, title: str, detail: str) -> JSONResponse:
    return JSONResponse(
        status_code=status,
        content={"type": "about:blank", "title": title,
                 "status": status, "detail": detail},
        headers={"Content-Type": "application/problem+json"},
    )

@router.get("/users/{user_id}")
async def get_user(user_id: str, user=Depends(verify_firebase_token)):
    if user.uid != user_id:
        return problem_detail(403, "Forbidden", "他ユーザーのデータへのアクセス不可")
    return {"id": user_id, "email": user.email}

@router.delete("/users/{user_id}", status_code=204)
async def delete_user(user_id: str, user=Depends(verify_firebase_token)):
    if user.uid != user_id:
        return problem_detail(403, "Forbidden", "Access denied")
    # 削除処理...
```

---

## トレードオフ

| 観点 | REST | GraphQL |
|---|---|---|
| キャッシュ | CDN活用しやすい | 困難（POSTが多い） |
| 学習コスト | 低い | 高い |
| Over-fetching | 発生しやすい | 解消できる |
| N+1問題 | 設計次第で回避可能 | DataLoader必須 |
| スキーマ管理 | OpenAPIで対応 | 型定義が強力 |

**URLバージョニング vs ヘッダーバージョニング**

- URLバージョニング: 可視性が高く運用しやすい → ほとんどのケースで推奨
- ヘッダーバージョニング: URLは綺麗だが、CDNキャッシュ設定・デバッグが複雑になる

---

## チェックリスト

- [ ] エラーレスポンスがRFC 7807（Problem Details）形式に統一されている
- [ ] URLに `/v1/` などのバージョンプレフィックスがある
- [ ] 全エンドポイントにFirebase IDトークン検証が設定されている
- [ ] レート制限が実装され、`X-RateLimit-*` ヘッダーを返している
- [ ] 破壊的変更（フィールド削除・型変更）はバージョンアップで対応する方針がある
