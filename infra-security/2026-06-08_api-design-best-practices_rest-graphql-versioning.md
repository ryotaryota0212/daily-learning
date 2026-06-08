# API設計のベストプラクティス：REST vs GraphQL・バージョニング・レート制限・認証

## 概要

APIはシステムの「契約」であり、一度公開すると変更コストが高い。  
AI時代においても「APIの設計力」はシステムを成立させる核心スキル。  
誤った設計は障害・セキュリティ事故・スケール問題に直結するため、  
「とりあえず動くエンドポイント」ではなく「壊れにくい契約設計」が求められる。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 適合ユースケース | リソース操作がシンプル、外部公開API | フロント主導のデータ取得、UI多様性が高い |
| Over/Under-fetch | 発生しやすい | 解消しやすい |
| キャッシュ | HTTP標準で容易（CDN対応） | 複雑（POSTがメイン、CDNが効きにくい） |
| スキーマ管理 | OpenAPI/Swaggerで補完 | スキーマファースト（型安全） |
| 学習コスト | 低 | 高（N+1問題、認可設計が難しい） |

**判断の結論**：
- 外部API・モバイルアプリ連携 → REST
- 社内BFF（Backend for Frontend）・ダッシュボード → GraphQL検討
- 迷ったらREST。GraphQLは問題が明確になってから導入する

### APIバージョニングの方式

- **URLパス方式**：`/api/v1/users` — 最も一般的、CDNキャッシュしやすい
- **ヘッダー方式**：`Accept: application/vnd.api.v2+json` — URLをクリーンに保てるが複雑
- **クエリパラメータ方式**：`?version=2` — 非推奨（キャッシュ汚染リスク）

**バージョン廃止戦略**：
- 最低2バージョン並存期間を設ける（通常6〜12ヶ月）
- `Sunset`ヘッダーで廃止日を明示
- 廃止予告はメール・ドキュメント・`Deprecation`ヘッダーで三重通知

### レート制限の設計

- **アルゴリズムの選択**：
  - Fixed Window：実装簡単、バースト攻撃に弱い
  - Sliding Window：精度高い、実装やや複雑
  - Token Bucket：バースト許容、API Gatewayで多用
- **キー設計**：IP単位 < ユーザーID単位 < APIキー単位（精度と管理コストのトレードオフ）
- **レスポンスヘッダー**：`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `Retry-After` を必ず返す

### 認証・認可の設計

- **認証**（誰か）：Firebase Auth JWT → Cloud Run でトークン検証
- **認可**（何ができるか）：エンドポイント単位のロールチェック + DB行レベル（RLS）
- **サービス間通信**：Cloud Run → Google-signed OIDC トークンを使用（APIキー禁止）

---

## アンチパターン vs 正しい設計

### アンチパターン

- `GET /api/getUser?action=delete` — HTTPメソッドの意味を無視
- バージョン管理なしで破壊的変更を本番に反映
- レート制限なしで全エンドポイントを公開
- エラーレスポンスに内部スタックトレースを含める
- 全エンドポイントで同じ認可ロジックを手書きコピペ

### 正しい設計

- HTTPメソッドを正しく使う：GET（取得・副作用なし）、POST（作成）、PUT/PATCH（更新）、DELETE（削除）
- 認可ロジックはミドルウェアに集約
- エラーは RFC 7807（Problem Details）形式で統一
- 破壊的変更は必ずバージョンを上げる

---

## コード例（FastAPI + Firebase Auth）

```python
from fastapi import Depends, HTTPException, Request
from fastapi.security import HTTPBearer
from firebase_admin import auth

security = HTTPBearer()

async def verify_token(credentials=Depends(security)):
    try:
        decoded = auth.verify_id_token(credentials.credentials)
        return decoded
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")

# レート制限ミドルウェア（Redis使用）
async def rate_limit(request: Request, user=Depends(verify_token)):
    key = f"rate:{user['uid']}"
    count = await redis.incr(key)
    if count == 1:
        await redis.expire(key, 60)
    if count > 100:  # 100req/min
        raise HTTPException(status_code=429, headers={"Retry-After": "60"})
    return user
```

---

## トレードオフ

| 選択肢 | メリット | コスト |
|--------|----------|--------|
| URLバージョニング | シンプル・CDN対応 | URL設計が冗長になる |
| GraphQL | 柔軟なデータ取得 | N+1問題・認可設計が複雑 |
| Token Bucket | バースト許容でUX良い | Redis等の状態管理が必要 |
| 厳格な認可ミドルウェア | セキュリティ強固 | 柔軟な権限設定が難しい |

---

## チェックリスト

- [ ] HTTPメソッドとステータスコードが仕様通りか（200/201/400/401/403/404/429）
- [ ] バージョニング戦略を決め、廃止プロセスを定義しているか
- [ ] レート制限がユーザーIDベースで実装され、適切なヘッダーを返しているか
- [ ] 認証・認可がミドルウェアに集約され、各エンドポイントで二重実装していないか
- [ ] エラーレスポンスに内部情報（スタックトレース・DBエラー）が漏れていないか
