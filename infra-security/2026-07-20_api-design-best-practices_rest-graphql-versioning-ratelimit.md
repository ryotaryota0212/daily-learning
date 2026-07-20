# API設計のベストプラクティス（REST vs GraphQL、バージョニング、レート制限、認証）

## 概要

AI時代においてAPIは「サービスの契約」そのものだ。コードは変えやすいが、公開済みAPIのブレイキングチェンジは利用者全員に影響する。
設計段階でバージョニング戦略・認証方式・レート制限を決めておかないと、後から修正するコストが爆発する。
「とりあえず動くエンドポイント」ではなく「壊れにくく、拡張しやすいAPI契約」を設計できることが重要。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いているケース | リソースが明確、モバイル以外 | 複雑な関係、モバイル帯域節約 |
| キャッシュ | HTTPキャッシュがそのまま使える | クエリ単位でキャッシュ設計が必要 |
| Over-fetching | フィールド絞り込みが難しい | クエリで必要フィールドのみ取得 |
| 学習コスト | 低い | 高い（スキーマ設計、N+1問題） |
| レート制限 | エンドポイント単位で簡単 | クエリの複雑度で計算が必要 |

**判断ルール（FastAPI + Cloud Run スタック）:**
- 社内・サービス間API → REST（シンプル、キャッシュ効く）
- 複数クライアント（Web/iOS/Android）が異なるフィールドを必要とする → GraphQL検討
- 迷ったらREST。後からGraphQLを追加する方が逆より容易

### バージョニング戦略

- **URLパスバージョニング** `/v1/users` → 最も直感的、採用率が高い
- **ヘッダーバージョニング** `API-Version: 2` → URLが綺麗だが、ブラウザからテスト難
- **クエリパラメータ** `?version=1` → 非推奨（キャッシュが効きにくい）

**実運用の原則:**
- 後方互換を維持できる変更（フィールド追加等）はバージョンアップ不要
- ブレイキングチェンジ（フィールド削除、型変更、レスポンス構造変更）でメジャーバージョンを上げる
- 旧バージョンは最低6ヶ月間はサポートし、廃止予告をDeprecationヘッダーで通知

### レート制限の設計

- **Fixed Window**: 実装簡単だが境界での burst（例：59秒と61秒で2倍リクエスト）が起きる
- **Sliding Window**: burst を防げるがメモリ消費大
- **Token Bucket**: バースト許容しつつ平均レートを制御できる。**本番推奨**

```python
# FastAPI + Redis でのToken Bucket実装例
import redis, time

r = redis.Redis()

def check_rate_limit(user_id: str, max_tokens: int = 100, refill_rate: float = 10.0) -> bool:
    key = f"rate:{user_id}"
    now = time.time()
    pipe = r.pipeline()
    pipe.hgetall(key)
    result = pipe.execute()[0]

    tokens = float(result.get(b"tokens", max_tokens))
    last_refill = float(result.get(b"last_refill", now))
    elapsed = now - last_refill
    tokens = min(max_tokens, tokens + elapsed * refill_rate)

    if tokens < 1:
        return False  # 429 Too Many Requests

    r.hset(key, mapping={"tokens": tokens - 1, "last_refill": now})
    r.expire(key, 3600)
    return True
```

### 認証・認可の設計

- **Firebase Auth + JWT**: Cloud Run環境では `Authorization: Bearer <token>` を検証
- ミドルウェアで全ルートに一括適用し、個別エンドポイントで細かい認可チェック
- APIキー（サービス間通信）は Secret Manager で管理し、環境変数に直書きしない

---

## アンチパターン vs 正しい設計

| アンチパターン | 問題 | 正しい設計 |
|---------------|------|-----------|
| `GET /deleteUser?id=1` | HTTPメソッドを無視した設計 | `DELETE /users/{id}` |
| エラーに200を返す | クライアントが判断できない | 適切なHTTPステータスコード（400/401/403/404/429/500）を使う |
| バージョンなしで本番公開 | ブレイキングチェンジ不可 | 最初から `/v1/` をつける |
| レスポンスに全フィールドを返す | 機密情報漏洩リスク | 必要なフィールドのみ明示的にシリアライズ |
| レート制限なし | DDoS/コスト爆発 | ユーザー・IP・APIキー単位で制限を設ける |

---

## レスポンス設計の原則

```json
// 成功レスポンス（統一フォーマット）
{
  "data": { "id": "123", "name": "Alice" },
  "meta": { "request_id": "uuid", "version": "v1" }
}

// エラーレスポンス
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Retry after 60s.",
    "retry_after": 60
  }
}
```

---

## トレードオフ

| 選択 | メリット | デメリット |
|------|---------|-----------|
| REST | シンプル、HTTPキャッシュ、ドキュメント容易 | Over-fetching、複数リクエスト問題 |
| GraphQL | 柔軟なクエリ、型安全 | N+1問題、キャッシュ複雑、複雑度制限が必要 |
| URLバージョニング | 明確、キャッシュ効く | URLが冗長 |
| 厳しいレート制限 | コスト・DDoS対策 | 正規ユーザーも影響を受ける可能性 |

---

## チェックリスト

- [ ] ブレイキングチェンジを含まない変更かどうかを確認してからデプロイする
- [ ] 全エンドポイントにレート制限を設け、429レスポンスに `Retry-After` ヘッダーを含める
- [ ] エラーレスポンスに内部スタックトレースや実装詳細を含めない
- [ ] APIキーや認証情報をレスポンスボディやURLパラメータに含めない
- [ ] Deprecationヘッダーで旧バージョン廃止を事前に通知する
