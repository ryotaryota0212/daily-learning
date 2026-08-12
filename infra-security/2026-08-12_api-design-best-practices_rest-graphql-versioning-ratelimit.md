# API設計のベストプラクティス（REST vs GraphQL・バージョニング・レート制限・認証）

## 概要

APIはシステムの「契約」であり、一度公開すると変更コストが高い。  
設計の失敗は将来の技術的負債に直結するため、**最初にトレードオフを理解して選択する力**が重要。  
AI時代においても「どのAPIをどう設計するか」の意思決定はエンジニアの核心スキルのまま。

---

## 仕組みの要点

### REST vs GraphQL の選択軸

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いているケース | リソースが明確、クライアント種類が少ない | クライアント多様、データ取得形状が様々 |
| キャッシュ | HTTP標準で容易 | クエリ毎に異なるため難しい |
| 型安全性 | OpenAPIで担保 | スキーマ定義で強制 |
| Over/Under-fetch | 起きやすい | クライアント制御できる |
| 学習コスト | 低い | 高い |

**結論：スタートアップ初期・小規模チームはREST。モバイル+Web+BFF構成になったらGraphQL検討。**

### バージョニング戦略

- **URLパス方式**（推奨）: `/api/v1/users` → シンプルで最も普及
- **ヘッダー方式**: `Accept: application/vnd.api.v1+json` → URLが綺麗だが複雑
- **クエリパラメータ**: `/api/users?version=1` → テストしやすいが非推奨

バージョンアップの原則:
- 後方互換性のある変更（フィールド追加）はバージョンを上げない
- 破壊的変更（フィールド削除・型変更）は必ずメジャーバージョンアップ
- 旧バージョンは最低6ヶ月は並行稼働してから廃止

### レート制限の設計

- **アルゴリズム選択**
  - Token Bucket: バースト許容、実装容易（推奨）
  - Sliding Window: 精度高いがRedis複雑
  - Fixed Window: シンプルだがウィンドウ境界で2倍バーストが起きうる
- **レベル設定**
  - エンドポイント別（書き込みは厳しく、読み込みは緩く）
  - ユーザー別・IPアドレス別・APIキー別
- **レスポンス**: `429 Too Many Requests` + `Retry-After` ヘッダー

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| `/getUser`, `/createUser`（動詞URL） | `GET /users/{id}`, `POST /users`（リソース+HTTPメソッド） |
| エラーを全て `200 OK` で返す | 適切なHTTPステータスコード使用 |
| バージョニングなしで破壊的変更 | v1/v2 で並行稼働して移行期間を設ける |
| 認証なしで内部APIを公開 | Cloud Run サービス間はIAM + IDトークン認証 |
| エラーにスタックトレースを返す | エラーIDのみ返し、詳細はサーバーログに |

---

## コード/設計例（FastAPI + Cloud Run）

```python
# レート制限 + 認証の最小実装例
from fastapi import FastAPI, Depends, HTTPException, Header
from google.auth.transport import requests
from google.oauth2 import id_token

app = FastAPI()

async def verify_firebase_token(authorization: str = Header(...)):
    token = authorization.replace("Bearer ", "")
    try:
        decoded = id_token.verify_firebase_token(
            token, requests.Request(), audience="YOUR_PROJECT_ID"
        )
        return decoded
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")

@app.get("/api/v1/users/{user_id}")
async def get_user(user_id: str, claims=Depends(verify_firebase_token)):
    if claims["uid"] != user_id:
        raise HTTPException(status_code=403, detail="Forbidden")
    return {"user_id": user_id}
```

レート制限はCloud Run の前段（Cloud Armor or Cloudflare）で実装するのがシンプル。

---

## トレードオフ

- **RESTの一貫性 vs GraphQLの柔軟性**: チームが小さいうちはRESTの学習コスト優位が大きい
- **バージョン管理の厳密さ vs 開発速度**: 初期は `/v1` だけ作り、変更が出たときに考える
- **レート制限の厳しさ vs UX**: 厳しすぎると正規ユーザーを弾く。最初はゆるく設定してログで調整
- **認証の安全性 vs レイテンシ**: JWTの検証は毎回ローカルで行いDBアクセス不要だがトークン無効化が即時でない

---

## チェックリスト

- [ ] HTTPメソッドとURIがRESTの原則に従っているか（動詞URLを使っていないか）
- [ ] バージョン変更ポリシーとdeprecation告知の仕組みがあるか
- [ ] レート制限が設定され `429` + `Retry-After` を返しているか
- [ ] エラーレスポンスにスタックトレースが含まれていないか
- [ ] サービス間通信はIAM認証（IDトークン）で保護されているか
