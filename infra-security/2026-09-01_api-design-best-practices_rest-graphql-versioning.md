# API設計のベストプラクティス — REST vs GraphQL・バージョニング・レート制限・認証

## 概要

APIはシステムの「契約」であり、一度公開すると変更コストが跳ね上がる。AI時代においてもAPIの設計品質はシステム全体の保守性・スケーラビリティに直結する。「とりあえず動くエンドポイント」ではなく、クライアントとの変更契約・セキュリティ境界・負荷特性を意識した設計が求められる。FastAPI + Cloud Run スタックでは特に、起動時間・認証フロー・レート制限の実装場所の判断が重要になる。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| クライアント種別 | 複数クライアントが固定エンドポイントを使う | クライアントごとに必要なフィールドが異なる |
| N+1問題 | URLごとにデータ固定→Over-fetch発生 | 1リクエストで必要分だけ取得可能 |
| キャッシュ | HTTP標準キャッシュが使いやすい | POST多用でHTTPキャッシュが難しい |
| 学習コスト | 低い（HTTP標準） | スキーマ設計・Resolver実装が必要 |
| 適合ケース | 外部公開API・シンプルなCRUD | BFF（Backend for Frontend）・ダッシュボード系 |

**判断の原則：** 外部公開・シンプルな操作はREST。内部BFF・複雑なフロントエンド要件はGraphQL。混在は保守コスト増のため避ける。

### バージョニング戦略

- **URLパス方式**（`/v1/users`）: 最も可視性が高い。リソース分離が明確
- **ヘッダー方式**（`Accept: application/vnd.api+json;version=2`）: URLクリーンだが、キャッシュ・テストが煩雑
- **クエリパラメータ方式**（`?version=1`）: 非推奨。ログ汚染・キャッシュキー複雑化

**推奨**: URLパス方式を採用し、v1が枯れたタイミングでv2を追加。v1は最低6ヶ月以上のdeprecation期間を設ける。

### レート制限の設計

- **実装場所**: Cloudflare WAF or Cloud Run 手前（APIゲートウェイ）が最適。アプリ層では遅すぎる
- **アルゴリズム**: Token Bucket（バースト許容）vs Fixed Window（シンプル）
- **識別子**: 認証済みユーザは `user_id`、未認証は `IP + User-Agent`
- **レスポンス**: `429 Too Many Requests` + `Retry-After` ヘッダー必須
- **Neon/DBへの直撃を防ぐ**: Redis or Cloud Memorystore でカウンタ管理

### 認証フロー（Firebase Auth + FastAPI）

1. クライアント → Firebase Auth → IDトークン取得
2. クライアント → FastAPI に `Authorization: Bearer <IDトークン>` 付与
3. FastAPI → Firebase Admin SDK でトークン検証（`verify_id_token`）
4. `uid` を取り出しDBのRLSコンテキストに渡す

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|--------------|------------|
| バージョンなしでAPI公開 | 初回から `/v1/` プレフィックス |
| アプリ層でレート制限実装 | エッジ/ゲートウェイ層で実装 |
| エラーに200 OKを返す | HTTPステータスコードを正しく使う |
| 認証トークンをURLパスに含める | Authorizationヘッダーで渡す |
| 全フィールドを常に返す | 必要なフィールドのみ（fields=パラメータ） |
| エラーメッセージに内部実装を露出 | 汎用エラーコード + ログで詳細管理 |

---

## コード/設計例（FastAPI + Firebase Auth）

```python
from fastapi import FastAPI, Depends, HTTPException, Header
from firebase_admin import auth, credentials, initialize_app

initialize_app(credentials.ApplicationDefault())
app = FastAPI()

async def get_current_user(authorization: str = Header(...)):
    scheme, token = authorization.split(" ", 1)
    if scheme.lower() != "bearer":
        raise HTTPException(status_code=401)
    try:
        decoded = auth.verify_id_token(token)
        return decoded["uid"]
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")

@app.get("/v1/users/me")
async def get_me(uid: str = Depends(get_current_user)):
    return {"uid": uid}

# レート制限はCloud Run手前のCloud Armorで設定
# terraform: google_compute_security_policy で rate_limit_options
```

---

## トレードオフ

| 選択 | メリット | デメリット |
|------|---------|-----------|
| REST（URLバージョン） | シンプル・キャッシュしやすい | エンドポイント数が増える |
| GraphQL | 柔軟・Over-fetch解消 | 複雑・キャッシュ難 |
| アプリ層レート制限 | 細かい制御可能 | DBへの攻撃を防げない |
| エッジ層レート制限 | DB保護・低レイテンシ遮断 | 細粒度の制御が難しい |
| Firebase Auth委任 | 実装コスト低・MFA対応 | Firebase依存・オフライン検証に工夫要 |

---

## チェックリスト

- [ ] APIは初回から `/v1/` プレフィックスを付与しているか
- [ ] レート制限をアプリより手前（Cloud Armor / Cloudflare）で実装しているか
- [ ] Firebase IDトークンの検証を毎リクエスト行い、期限切れを適切に処理しているか
- [ ] エラーレスポンスに内部スタックトレースや実装詳細が露出していないか
- [ ] `429` レスポンスに `Retry-After` ヘッダーを付与しているか
