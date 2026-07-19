# API設計のベストプラクティス — REST vs GraphQL、バージョニング、認証

## 概要

AI時代においてAPIは「システムの接合部」であり、設計の善し悪しがスケーラビリティ・保守性・セキュリティに直結する。  
「とりあえず動くエンドポイント」ではなく、**変更に強く、壊れにくく、意図が伝わるAPI**を設計する力が求められる。  
FastAPI + Cloud Run + Firebase Auth のスタックでは、設計上の判断が後のインフラコストや障害対応の難易度に直接影響する。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いている用途 | リソース単位の操作、公開API | 複雑な関連取得、フロント主導 |
| over-fetch/under-fetch | 起きやすい | クライアントが制御 |
| キャッシュ | HTTP標準で容易 | 工夫が必要（Persisted Query等） |
| スキーマ変更 | バージョニングが必要 | 非破壊的に追加可能 |
| モニタリング | シンプル | クエリ複雑度の監視が必要 |

**FastAPI + Cloud Run スタックの推奨**: 初期はREST、フロントが複数かつ取得パターンが多様になったらGraphQL検討。

### バージョニング戦略

- **URLパス方式** (`/v1/`, `/v2/`): 最も明示的、キャッシュも容易。推奨。
- **ヘッダー方式** (`Accept: application/vnd.api+json;version=2`): URLが綺麗だが運用が複雑。
- **廃止ポリシー**: 旧バージョンは最低6ヶ月維持し、`Sunset`ヘッダーで予告。

### 認証・認可設計

- Firebase Auth → IDトークン取得 → `Authorization: Bearer <token>` で送信
- Cloud Run側でトークン検証（Firebaseの公開鍵で署名確認）
- エンドポイントごとに**認証(誰か)と認可(何ができるか)を分離**して実装

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|----------------|------------|
| `/getUser`, `/createUser` (動詞URL) | `GET /users/{id}`, `POST /users` (名詞+HTTPメソッド) |
| エラーを全部200で返す | 4xx/5xx を適切に使い、エラー詳細はbodyに |
| バージョニングなし | `/api/v1/` から始め、破壊的変更時にインクリメント |
| 認証トークンをURLクエリに含める | `Authorization` ヘッダーのみ（ログに残るため） |
| 全フィールドを毎回返す | `fields=id,name` のような絞り込みを許容する |

---

## コード例（FastAPI + Firebase Auth）

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import firebase_admin.auth as firebase_auth

security = HTTPBearer()

async def verify_token(
    credentials: HTTPAuthorizationCredentials = Depends(security)
) -> dict:
    try:
        return firebase_auth.verify_id_token(credentials.credentials)
    except Exception:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED)

@app.get("/api/v1/users/me")
async def get_me(user: dict = Depends(verify_token)):
    return {"uid": user["uid"], "email": user.get("email")}
```

---

## トレードオフ

- **REST の明示性 vs GraphQL の柔軟性**: チームが小さく要件が安定しているうちはRESTが低コスト
- **バージョニングのコスト**: URL versioning は運用明確だが、旧バージョンの維持コストが積む
- **認証の厳密さ vs 開発速度**: 全エンドポイントに認証を入れると初期は遅いが、後から足すよりはるかに安全
- **レート制限**: Cloud RunはCloud Armor/API Gatewayで実施。アプリ層では実装しない方がシンプル

---

## チェックリスト

- [ ] エンドポイントは名詞+HTTPメソッドで設計し、動詞URLを排除した
- [ ] 認証はすべて `Authorization: Bearer` ヘッダー経由で行い、URLに含めない
- [ ] `/api/v1/` のようにバージョンをURLに含め、破壊的変更時のポリシーを決めた
- [ ] エラーレスポンスに適切なHTTPステータスコードと構造化メッセージを返している
- [ ] Cloud Armor or Cloudflare でレート制限を設定し、アプリ層への負荷を減らした
