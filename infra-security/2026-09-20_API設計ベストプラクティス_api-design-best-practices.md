# API設計のベストプラクティス

## 概要
AIが「動くコード」を生成できる時代、API設計の真価は「変化に強く、障害が伝播しない境界」を作れるかどうかにある。設計ミスはサービス全体の可用性・セキュリティ・スケーラビリティに直結する。FastAPI + Firebase Auth + Cloud Run スタックでは特に認証境界・バージョニング・レート制限の設計が重要。

---

## 仕組みの要点

### REST vs GraphQL の選択基準
- **REST**: リソースが明確・キャッシュ重要・チームが小さい → デフォルト選択
- **GraphQL**: 複雑なリレーション・多様なクライアント（web/mobile）・過剰フェッチが問題 → N+1問題対策必須
- FastAPI + Neon 構成では **REST が推奨**（SQLクエリ制御が容易、OpenAPI自動生成のメリット大）

### バージョニング戦略
- **URLパス方式**: `/v1/users` → 最もシンプル・推奨
- **ヘッダー方式**: `API-Version: 2` → URLが綺麗だがキャッシュ制御が複雑
- 後方互換を維持しながら新バージョンを追加する（旧バージョンを突然廃止しない）
- Deprecation予告は最低3ヶ月前にレスポンスヘッダーで通知: `Sunset: Sat, 20 Sep 2027 00:00:00 GMT`

### 認証設計
- Firebase Auth の JWT を `Authorization: Bearer <token>` で受け取る
- `uid` をユーザー識別子として使用（メールは変更可能なので不使用）
- サービス間通信は Cloud Run のサービスアカウントで認証（JWTではなくADCを使用）
- 公開エンドポイントと認証必須エンドポイントを明示的に分離する

### レート制限
- エンドポイント単位でリミットを設定（全体一律は粗すぎる）
- ヘッダーで状態を返す: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`
- 超過時は `429 Too Many Requests` を返す
- Cloud Run + Upstash Redis（または Cloud Armor）で実装

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| エラーでも常に 200 を返す | HTTPステータスコードを正しく使う（400/401/403/404/500） |
| `/getUser`, `/createUser` のような動詞URL | `/users` (GET一覧・POST作成)、`/users/{id}` (GET/PUT/DELETE) |
| レスポンス構造がエンドポイントごとに違う | 統一レスポンス形式 `{data, error, meta}` |
| バージョニングなしで Breaking Change | `/v1/` → `/v2/` で段階移行 |
| 内部エラーをそのまま返す | エラーは `{code, message}` のみ。スタックトレース非公開 |

---

## コード例（FastAPI）

```python
from fastapi import APIRouter, Depends, HTTPException, status

router = APIRouter(prefix="/v1")

def ok(data, meta=None):
    return {"data": data, "error": None, "meta": meta}

@router.get("/users/{user_id}")
async def get_user(user_id: str, uid: str = Depends(verify_firebase_token)):
    if uid != user_id:
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN,
                            detail={"code": "FORBIDDEN", "message": "Access denied"})
    user = await db.get_user(user_id)
    if not user:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND,
                            detail={"code": "NOT_FOUND", "message": "User not found"})
    return ok(user)
```

---

## トレードオフ

| 選択 | メリット | デメリット |
|---|---|---|
| REST | シンプル・キャッシュ容易・OpenAPI自動生成 | 過剰フェッチが起きやすい |
| GraphQL | 柔軟・過剰フェッチ回避・1リクエストで複数取得 | N+1問題・複雑な認証制御・キャッシュ困難 |
| URLバージョニング | 直感的・キャッシュ制御しやすい | URLが長くなる |
| ヘッダーバージョニング | URLが綺麗 | ルーティング複雑・デバッグしにくい |

---

## チェックリスト
- [ ] HTTPステータスコードが正しく使われているか（200/201/400/401/403/404/500）
- [ ] 認証必須エンドポイントで Firebase Auth の JWT 検証が実装されているか
- [ ] エラーレスポンスに内部情報（スタックトレース・DB詳細）が含まれていないか
- [ ] レート制限が設定され、`X-RateLimit-*` ヘッダーを返しているか
- [ ] Breaking Change 時にバージョニングで段階移行できる設計になっているか
