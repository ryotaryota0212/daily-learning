# API設計のベストプラクティス（REST vs GraphQL、バージョニング、レート制限、認証）

## 概要

APIはシステムの境界面であり、「どう設計するか」はスケーラビリティ・保守性・セキュリティのすべてに直結する。  
「とりあえず動くエンドポイント」ではなく、変更に強く、クライアントに優しく、障害に強いAPIを設計する力がAI時代のエンジニアには求められる。  
FastAPI + Cloud Run 環境での判断基準と実践パターンを整理する。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いているケース | リソース単位が明確、公開API | 複雑な関係、モバイル・BFF |
| キャッシュ | URLベースで容易 | POST中心で難しい |
| 学習コスト | 低い | 高い |
| スキーマ型安全 | OpenAPI別途必要 | 組み込み |
| N+1問題 | 起きにくい | DataLoaderで対処必要 |

**結論**: 小〜中規模のSaaSはRESTが無難。複数クライアント（Web/モバイル）で取得データが違う場合はGraphQLが有効。

### バージョニング戦略

- **URLパス方式**: `/api/v1/users` → 最も明示的で安全
- **ヘッダー方式**: `API-Version: 2026-08-01` → Stripe等が採用
- **クエリパラメータ**: `/users?version=2` → 避けるべき（キャッシュが効かない）

**推奨**: FastAPIでは `/api/v1/` プレフィックスをルーターで分ける。

### レート制限の設計

- **スライディングウィンドウ**: 直近N秒を常に見る（精度高い）
- **固定ウィンドウ**: 実装簡単だがウィンドウ境界で2倍のリクエストが通る
- **トークンバケット**: バースト許容しつつ平均を制限（推奨）
- レート制限ヘッダーを返す: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`
- 429エラーには `Retry-After` ヘッダーをセットする

### 認証フロー（Firebase Auth + FastAPI）

- クライアントがFirebase IDトークン取得 → `Authorization: Bearer <token>` でAPI呼び出し
- FastAPIがトークン検証（firebase-admin SDK）→ `uid` をリクエストコンテキストに付与
- DBのRLSに `uid` を渡して行レベル制御

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| `/getUser`, `/createUser` → 動詞URL | `/users`, `/users/{id}` → 名詞 + HTTPメソッド |
| エラーを全部200で返す | 4xx/5xx を適切に使い分ける |
| レスポンスに内部エラー詳細を返す | 本番では汎用メッセージ、詳細はログに |
| 破壊的変更をバージョン無しで行う | v2エンドポイントを追加、v1は維持 |
| 認証なしでIDを連番整数 | UUIDv4 + 認証チェック必須 |
| ページネーション未実装 | カーソルベースor offset+limitを最初から設計 |

---

## コード例（FastAPI + Firebase Auth）

```python
from fastapi import Depends, HTTPException, Security
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import firebase_admin.auth as fb_auth

security = HTTPBearer()

async def get_current_user(creds: HTTPAuthorizationCredentials = Security(security)):
    try:
        decoded = fb_auth.verify_id_token(creds.credentials)
        return decoded  # uid, email など含む
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")

# レート制限ミドルウェア（Redis利用）
from fastapi import Request
from redis.asyncio import Redis

async def rate_limit(request: Request, redis: Redis):
    key = f"rl:{request.client.host}"
    count = await redis.incr(key)
    if count == 1:
        await redis.expire(key, 60)  # 60秒ウィンドウ
    if count > 100:  # 100req/min
        raise HTTPException(status_code=429, headers={"Retry-After": "60"})
```

---

## トレードオフ

| 設計判断 | メリット | デメリット |
|---|---|---|
| REST | シンプル、キャッシュ容易 | 多エンドポイント増加 |
| GraphQL | 柔軟、過不足取得なし | 複雑、キャッシュ困難 |
| URLバージョニング | 明示的、テスト容易 | URL長くなる |
| カーソルページネーション | リアルタイム更新でもズレない | 実装複雑 |
| レート制限(Redis) | 正確、分散対応 | Redis依存、単一障害点 |

**重要な判断**: レート制限はCloud RunのインスタンスごとにではなくRedis等共有ストアで管理しないと、スケールアウト時に意味をなさない。

---

## チェックリスト

- [ ] エンドポイントは名詞 + HTTPメソッドで設計されているか（動詞URLではないか）
- [ ] バージョニング戦略が決まっており、破壊的変更のルールがあるか
- [ ] レート制限がインスタンスローカルではなく共有ストア（Redis等）で管理されているか
- [ ] 認証トークン検証が全エンドポイントで行われているか（デフォルト拒否）
- [ ] エラーレスポンスに内部実装の詳細が漏れていないか
