# API設計のベストプラクティス（REST vs GraphQL、バージョニング、レート制限、認証）

## 概要

APIはシステムの「契約」であり、一度公開すると変更コストが高い。
設計の誤りはクライアント側への破壊的変更や、セキュリティ上の穴として長期間残る。
AI時代でも「何を公開してどう守るか」を決めるのは人間であり、API設計力はシステム全体を左右する。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いているケース | リソース単位のCRUD、公開API | 複雑なデータ取得、BFF層、フロント主導 |
| オーバーフェッチ | 発生しやすい | 必要なフィールドだけ取得可能 |
| キャッシュ | HTTP標準（CDN対応容易） | 難しい（POSTベースのため） |
| 型安全性 | OpenAPIで補完 | スキーマが型の源泉 |
| レート制限実装 | URLベースで容易 | クエリ深度・コスト計算が必要 |

**結論：FastAPI + Cloud Run 構成では REST を基本とし、複雑なデータ要求がある場合のみ GraphQL を検討。**

### APIバージョニング戦略

- **URLパス方式**（推奨）: `/v1/users`, `/v2/users`
  - CDNやロードバランサでルーティングしやすい
  - 旧バージョンを独立したサービスとして維持できる
- **ヘッダー方式**: `Accept: application/vnd.myapi.v2+json`
  - URLがシンプルだが、ブラウザのキャッシュ管理が難しい
- **クエリパラメータ方式**: `?version=2`（非推奨）
  - テスト向けのみ。本番では避ける

### レート制限の設計

- **単位**：ユーザーID or APIキー or IPアドレス
- **アルゴリズム**：
  - Token Bucket（バースト許容）→ 通常の用途
  - Fixed Window（実装シンプル）→ 粗いアクセス制御
  - Sliding Window（精度高い）→ 重要エンドポイント
- **バックエンド**：Redis（TTLと原子カウンタ）を推奨
- **レスポンスヘッダーで状態通知**：
  ```
  X-RateLimit-Limit: 1000
  X-RateLimit-Remaining: 42
  X-RateLimit-Reset: 1726142400
  Retry-After: 60  ← 429時のみ
  ```

### API認証パターン

- **Firebase Auth（Bearer Token）**：ユーザー操作系API
- **APIキー**：サービス間、外部クライアント向け（ヘッダー推奨: `X-API-Key`）
- **mTLS**：Cloud Run サービス間の内部通信
- **IP制限**：管理APIへの追加レイヤー

---

## アンチパターン vs 正しい設計

### アンチパターン

```python
# Bad: 認証なしでユーザーIDを直接受け取る
@app.get("/users/{user_id}/data")
async def get_data(user_id: str):
    return db.fetch(user_id)  # 他人のデータも取れる
```

- バージョン管理なし → 破壊的変更でクライアントが壊れる
- エラーレスポンスがバラバラ（文字列、ネストJSONが混在）
- レート制限なし → スクレイピングや意図しない高負荷に無防備
- 全エラーに 500 を返す → クライアントが原因を判断できない

### 正しい設計

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer

security = HTTPBearer()

@app.get("/v1/users/me/data")
async def get_my_data(token = Depends(security), db = Depends(get_db)):
    user = verify_firebase_token(token.credentials)
    # 自分のデータのみ返す（RLSとも連動）
    return await db.fetch_user_data(user.uid)
```

**統一エラーフォーマット**:
```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Retry after 60 seconds.",
    "retry_after": 60
  }
}
```

---

## トレードオフ

| 選択 | メリット | デメリット |
|------|----------|------------|
| URLバージョニング | 明示的、CDN対応 | URLが増える |
| Token Bucket | バースト対応 | Redis必須、実装複雑 |
| GraphQL | 柔軟なデータ取得 | キャッシュ・レート制限が難 |
| APIキー認証 | シンプル | 漏洩時の影響が大きい |

**補足**：レート制限のRedisは Cloud Memorystore（CloudRun同一VPC）を使い、外部公開しない。

---

## チェックリスト

- [ ] エンドポイントには `/v1/` などバージョンプレフィックスを付与している
- [ ] 全APIに認証を実装し、自分のリソースのみアクセスできる設計になっている
- [ ] レート制限を実装し、`429` レスポンスに `Retry-After` を含めている
- [ ] エラーレスポンスを統一したフォーマットで返している
- [ ] 管理系エンドポイントは追加のIP制限またはmTLSで保護している
