# API設計のベストプラクティス（REST vs GraphQL・バージョニング・レート制限・認証）

## 概要

APIはシステムの「契約」であり、一度公開すると変更コストが極めて高い。  
AI時代においても「何を・どう公開するか」の設計判断は人間に求められる核心的なスキル。  
FastAPI + Firebase Auth のスタックでは、認証・バージョニング・レート制限を最初から設計に組み込むことが壊れにくいAPIを作る鍵になる。

---

## 仕組みの要点

### REST vs GraphQL の選択基準

| 観点 | REST | GraphQL |
|------|------|---------|
| 向いている場面 | リソースが明確・クライアントが少数 | クライアントが多様・フィールド取得量を最適化したい |
| N+1問題 | エンドポイント設計で回避 | DataLoader 必須 |
| キャッシュ | HTTP キャッシュが使いやすい | 複雑（クエリハッシュ単位） |
| 学習コスト | 低い | 高い |
| **FastAPI との相性** | ◎ 公式サポート | △ strawberry 等が必要 |

**判断原則**: 社内API・スタートアップ初期はREST。BFF（Backend for Frontend）パターンでクライアント別にデータ形状が異なる場合はGraphQLを検討。

---

### バージョニング戦略

- **URLパスバージョニング** `/v1/users` → 最も明確、FastAPIで管理しやすい
- **ヘッダーバージョニング** `API-Version: 2024-01-01` → URLを汚さないが実装複雑
- **非推奨パラメータ** `?version=2` → キャッシュ汚染リスクあり

**推奨**: URLパスで `/v1/`, `/v2/` を FastAPI の APIRouter で分離する。

---

### レート制限の設計

- **目的**: DDoS対策・コスト保護・公平な利用確保
- **単位**: IP単位 < ユーザー単位 < APIキー単位（粒度が細かいほど正確）
- **アルゴリズム**: Fixed Window（単純）/ Sliding Window（正確）/ Token Bucket（バースト許容）
- **レスポンスヘッダー** で残量を伝える:
  - `X-RateLimit-Limit: 100`
  - `X-RateLimit-Remaining: 47`
  - `X-RateLimit-Reset: 1720000000`

---

### 認証の組み込み

- Firebase Auth の JWT を `Authorization: Bearer <token>` で受け取る
- FastAPI の `Depends()` で全エンドポイントに強制適用
- 認証なしエンドポイントは明示的に許可リストで管理する（デフォルト拒否）

---

## アンチパターン vs 正しい設計

### アンチパターン

- バージョニングなしで破壊的変更をデプロイ → クライアントが突然壊れる
- 認証チェックをルートごとに手動で実装 → 漏れが発生する
- レート制限なし → 悪意あるユーザーがコストを食い潰す
- 全エラーを `500 Internal Server Error` で返す → クライアントがリトライ戦略を取れない
- レスポンスに内部スタックトレースを含める → 攻撃者に構造を晒す

### 正しい設計

- 認証は Middleware / Depends で一元管理
- エラーは意味のあるHTTPステータスコード + 構造化エラーレスポンス
- バージョンを URL に含め、旧バージョンは非推奨期間を設けてから廃止
- レート制限は Redis + Sliding Window で実装し、超過時は `429 Too Many Requests`

---

## コード例（FastAPI + Firebase Auth）

```python
from fastapi import FastAPI, Depends, HTTPException, Request
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import firebase_admin
from firebase_admin import auth

app = FastAPI()
security = HTTPBearer()

async def verify_token(creds: HTTPAuthorizationCredentials = Depends(security)):
    try:
        decoded = auth.verify_id_token(creds.credentials)
        return decoded
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")

# v1ルーター（認証必須をDependsで強制）
from fastapi import APIRouter
v1 = APIRouter(prefix="/v1", dependencies=[Depends(verify_token)])

@v1.get("/users/me")
async def get_me(user=Depends(verify_token)):
    return {"uid": user["uid"]}

app.include_router(v1)
```

---

## トレードオフ

| 選択 | メリット | デメリット |
|------|---------|-----------|
| URLバージョニング | 明確・シンプル | URL が長くなる |
| 厳格なレート制限 | コスト保護・公平性 | 正規ユーザーを誤ってブロックするリスク |
| GraphQL採用 | 柔軟なデータ取得 | N+1・キャッシュ・学習コスト増 |
| 認証をMiddlewareに集約 | 漏れがなくなる | 一部パスの除外設定が必要 |

---

## チェックリスト

- [ ] 全エンドポイントにバージョンプレフィックスが付いているか
- [ ] 認証チェックが `Depends()` で強制されており、手動呼び出しに依存していないか
- [ ] レート制限が実装され、`429` と残量ヘッダーを返しているか
- [ ] エラーレスポンスにスタックトレース・内部パスが含まれていないか
- [ ] 破壊的変更は新バージョンで提供し、旧バージョンに非推奨通知を出しているか
