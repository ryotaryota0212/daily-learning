# マイクロサービス vs モノリスの判断基準と移行戦略

## 概要

「マイクロサービスにすべきか？」はAI時代のシステム設計でも最頻出の意思決定だ。正解は「チームとドメインの複雑さに依存する」。多くのスタートアップがモノリスで十分なのに早期にマイクロサービス化し、運用コストで失速する。逆に大規模組織がモノリスに縛られてデプロイが週1回になる。この判断を根拠を持って下せることがAI時代のエンジニアに求められる。

---

## 仕組みの要点

### モノリス（Monolith）
- 単一デプロイ単位。1プロセスで全機能が動く
- DB、キャッシュ、ビジネスロジックが密結合
- 開発初期は圧倒的に速い。デバッグ・テストが容易
- スケールの限界：特定機能だけスケールできない

### マイクロサービス（Microservices）
- 機能ごとに独立したサービス。それぞれ個別デプロイ可能
- サービス間通信はHTTP/gRPC、または非同期メッセージング
- スケール・デプロイ・障害が独立する
- 分散システムの複雑さ（ネットワーク遅延、データ一貫性、分散トレーシング）が伴う

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい判断 |
|---|---|
| チーム2〜3人なのにマイクロサービス | チームが小さい間はモノリスで速度優先 |
| 「なんとなくモダン」でマイクロサービス化 | ドメイン境界が明確になってから分割 |
| 巨大モノリスを一気に分割 | Strangler Figパターンで段階的移行 |
| サービス間でDBを共有 | サービスごとにDBを所有（Shared DBはNG） |
| 分散モノリス（呼び出しが密結合） | サービスが独立してデプロイ可能な状態を維持 |

### 判断基準チェック（モノリスを続けるべき条件）
- チームが10人以下
- ドメインの境界がまだ不明確
- デプロイ頻度が週数回以下
- スケール要件が単一サービスで対応可能

---

## コード/設計例：Strangler Figパターン（段階的移行）

モノリスの一部機能をCloud Runサービスとして切り出す例。

```python
# FastAPI: 新マイクロサービス（通知サービスを切り出し）
# 旧モノリスからはHTTP経由で呼ぶ

from fastapi import FastAPI, Depends
from google.auth.transport.requests import Request
from google.oauth2 import id_token

app = FastAPI()

async def verify_internal(token: str):
    # Cloud Run サービス間認証（OIDC）
    id_token.verify_oauth2_token(token, Request())

@app.post("/notify")
async def send_notification(payload: dict, _=Depends(verify_internal)):
    # 通知ロジックをここに集約
    return {"status": "sent"}
```

```yaml
# Cloud Run: 内部のみ（外部公開しない）
# ingress: internal により直接アクセス不可
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: notification-service
spec:
  template:
    metadata:
      annotations:
        run.googleapis.com/ingress: internal
```

---

## トレードオフ

| 観点 | モノリス | マイクロサービス |
|---|---|---|
| 開発速度（初期） | 速い | 遅い（設定・CI/CD複数） |
| デプロイ独立性 | 全体再デプロイ必要 | 個別デプロイ可能 |
| スケーラビリティ | 全体スケールのみ | 機能別スケール可能 |
| 障害影響範囲 | 全体に波及しやすい | サービス単位で局所化 |
| 運用複雑度 | 低い | 高い（分散トレーシング必須） |
| データ一貫性 | 容易（同一DB） | 難しい（Saga/イベント設計が必要） |

**FastAPI + Cloud Run スタックでの推奨：**
- 初期フェーズ：単一Cloud Runサービス（モノリス）で十分
- 特定機能のスケール要件が出たら：その機能だけ切り出す（Strangler Fig）
- 各サービスは Neon の別スキーマ or 別DBインスタンスを持つ

---

## チェックリスト

- [ ] チームサイズと機能数を見て「今モノリスで十分か」を問い直した
- [ ] マイクロサービス化する場合、ドメイン境界（Bounded Context）が明確になっている
- [ ] サービス間でDBを共有していない（Shared DBアンチパターンを避けた）
- [ ] Cloud Run サービス間通信はOIDCトークンで認証している
- [ ] 段階移行（Strangler Fig）の計画があり、一気に全移行しない方針になっている
