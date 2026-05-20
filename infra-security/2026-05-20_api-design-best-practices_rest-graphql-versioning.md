# API設計のベストプラクティス（REST vs GraphQL・バージョニング・認証）

## 概要

APIはシステムの「契約」であり、一度公開すると変更コストが高い。
設計の判断（REST か GraphQL か、バージョニング方式、認証方式）は後から変えにくく、
スケール・セキュリティ・開発体験に長期間影響し続ける。
「とりあえず動くエンドポイント」ではなく「壊れにくく進化できるAPI」を設計する力が問われる。

---

## 仕組みの要点

### REST vs GraphQL の判断基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 適合ユースケース | CRUD中心・公開API | 複雑な関連データ・BFF層 |
| キャッシュ | CDN/HTTPキャッシュが容易 | POSTが多くキャッシュ困難 |
| 過剰/不足取得 | 起きやすい | クライアント指定で解決 |
| 型安全 | OpenAPI（手動） | スキーマが型定義そのもの |
| 学習コスト | 低 | 高（N+1問題、DataLoader等） |

**判断指針：**
- 公開API・CDNキャッシュ重視 → REST
- 複数クライアント（Web/Mobile）でデータ形状が異なる → GraphQL or BFF
- 小規模スタートアップ → RESTで十分

### バージョニング戦略

- **URLパス方式**（`/v1/users`）: 最も明示的、CDNルーティングも簡単。推奨
- **ヘッダー方式**（`Accept: application/vnd.api+json;version=2`）: URL汚染なし、デバッグが面倒
- **クエリパラメータ**（`?version=2`）: キャッシュキーが複雑になる。避ける

**後方互換の原則：**
- フィールド追加は非破壊（許容）
- フィールド削除・型変更は破壊的変更（バージョン上げ必須）
- 廃止予定フィールドは `Deprecation` ヘッダーで通知 → 一定期間後削除

### 認証方式の選択

- **Firebase Auth + JWT**: 小規模SaaS向け。Cloud RunはIDトークンをミドルウェアで検証
- **API Key**: Machine-to-machine。シークレットローテーション設計が必須
- **OAuth2 Client Credentials**: サービス間認証の標準

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|--------------|----------|
| `/getUser`, `/createUser` (動詞URL) | `GET /users/{id}`, `POST /users` (リソース+メソッド) |
| エラーも常に `200 OK` 返す | 適切なHTTPステータス（400/401/403/404/422/500） |
| バージョンなしで破壊的変更 | `/v1/` で明示、変更時は `/v2/` |
| 認証トークンをURLに含める | `Authorization: Bearer` ヘッダー使用 |
| レスポンスに不要な内部情報を含める | 必要なフィールドのみ返す（情報漏洩防止） |

---

## コード例（FastAPI + Firebase Auth）

```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import firebase_admin
from firebase_admin import auth

app = FastAPI()
bearer = HTTPBearer()

def verify_token(cred: HTTPAuthorizationCredentials = Depends(bearer)):
    try:
        decoded = auth.verify_id_token(cred.credentials)
        return decoded
    except Exception:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED)

# v1エンドポイント: 後方互換を維持したまま v2 で拡張可能
@app.get("/v1/users/{user_id}")
def get_user(user_id: str, user=Depends(verify_token)):
    if user["uid"] != user_id:
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN)
    return {"id": user_id, "email": user["email"]}
```

---

## トレードオフ

- **REST の細粒度エンドポイント**: クライアントの柔軟性は低いが、キャッシュ・モニタリングが単純
- **GraphQL の柔軟性**: 開発体験は良いが、N+1問題・認可の複雑さ・クエリ深度制限が必要
- **URLバージョニング**: 運用シンプル、但し古いバージョンの保守コストが増える
- **ヘッダーバージョニング**: URLは綺麗だが、curl/ブラウザでのテストが面倒

---

## チェックリスト

- [ ] エンドポイントはリソース名（名詞）＋HTTPメソッドで設計されているか
- [ ] 全エンドポイントに認証ミドルウェアが適用されているか（認証不要なエンドポイントは明示的に除外）
- [ ] エラーレスポンスに内部スタックトレースや実装詳細が含まれていないか
- [ ] `/v1/` プレフィックスがあり、破壊的変更時にバージョンを上げる運用になっているか
- [ ] レート制限（Cloud Run の同時実行数 + APIゲートウェイの制限）が設定されているか
