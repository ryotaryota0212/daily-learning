# API設計のベストプラクティス：REST vs GraphQL・バージョニング・認証

## 概要

APIは「外部との契約」であり、一度公開すると変更コストが高い。AI時代においても、LLMがコードを生成するにはまず**何をどう呼べばよいか**が明確に設計されている必要がある。「とりあえず動くエンドポイント」を並べるのではなく、一貫した設計原則・バージョニング戦略・セキュリティ境界を備えたAPIが、長期的な開発速度と安全性を決定する。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いているケース | 公開API・シンプルなCRUD・キャッシュ重要 | 複雑な関連データ・BFF・モバイル |
| キャッシュ | HTTP標準（CDN対応しやすい） | クエリ単位（CDN難しい） |
| 型安全 | OpenAPI/Swagger で補完 | スキーマがあり強い |
| Over/Under Fetch | 起きやすい | 解決できる |
| 運用コスト | 低い | 高い（スキーマ管理、N+1問題） |

**FastAPIスタックの推奨：基本はREST + OpenAPI。複雑なネスト関係が多い場合のみGraphQL検討。**

### バージョニング戦略

- **URLパス方式**（`/v1/users`）：シンプル・可視性高い。推奨
- **ヘッダー方式**（`Accept: application/vnd.api+json;version=1`）：URL汚染なし・テスト煩雑
- **クエリパラメータ方式**（`?version=1`）：非推奨。キャッシュ複雑化

**後方互換の変更（フィールド追加等）はバージョンアップ不要。破壊的変更のみ `/v2/` を切る。**

### レート制限の設計

- 識別子は `user_id > API Key > IP` の優先順で使う
- スライディングウィンドウ方式が公平（固定ウィンドウはバースト許容してしまう）
- 429レスポンスには `Retry-After` ヘッダーを必ず返す
- エンドポイント種別ごとに閾値を変える（書き込み < 読み取り）

### API認証の設計

- **ユーザー向け**：Firebase Auth の JWTトークンを Bearer で受け取り、Cloud Run 側で検証
- **サービス間**：Google ID Token（サービスアカウント） or Cloud Run の組み込み認証
- **外部パートナー向け**：API Key + HMAC署名（リプレイ攻撃対策）

---

## アンチパターン vs 正しい設計

### アンチパターン

```
# 悪い例
POST /api/getUser          # 動詞をURLに入れる
GET  /api/delete_post/123  # GETで副作用
POST /api/v1/users         # エラー時も200で返す（body内にerror）
```

- レスポンス形式が一貫しない（ある時は `{data: ...}`、ある時は配列直返し）
- エラー詳細をそのまま返す（スタックトレース、DBエラーメッセージ）
- バージョンなしで公開したまま仕様変更する

### 正しい設計

```python
# FastAPI の例：統一レスポンス + 適切なHTTPステータス
from fastapi import APIRouter, HTTPException, status

router = APIRouter(prefix="/v1")

@router.get("/users/{user_id}", response_model=UserResponse)
async def get_user(user_id: str, current_user: User = Depends(get_current_user)):
    user = await db.get_user(user_id)
    if not user:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND,
                           detail="User not found")  # 内部情報を含めない
    return user  # 200 OK

@router.post("/users", response_model=UserResponse,
             status_code=status.HTTP_201_CREATED)
async def create_user(body: CreateUserRequest):
    ...
```

---

## トレードオフ

| 選択 | メリット | デメリット |
|------|---------|-----------|
| URLバージョニング | シンプル・CDN friendly | URLが増える |
| 厳格な型検証（Pydantic） | バグ早期発見 | 柔軟性低下・移行コスト |
| レート制限をAPI GW側に任せる | コードシンプル | GW依存・細かい制御不可 |
| GraphQL | 型安全・柔軟 | N+1問題・キャッシュ困難 |

---

## チェックリスト

- [ ] HTTPメソッドとステータスコードが正しく使われているか（GETは副作用なし、201/204を使い分け）
- [ ] エラーレスポンスにスタックトレースやDB情報が含まれていないか
- [ ] 破壊的変更時に `/v2/` を切り、`/v1/` の廃止スケジュールをドキュメント化しているか
- [ ] レート制限が設定され、429時に `Retry-After` ヘッダーを返しているか
- [ ] サービス間通信でサービスアカウント + Google ID Token を使っているか（API Keyの使い回し禁止）
