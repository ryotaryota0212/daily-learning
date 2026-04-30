# 可用性設計：SLA/SLO・冗長化・フェイルオーバー・グレースフルデグラデーション

## 概要

「動くシステム」と「壊れにくいシステム」を分けるのが可用性設計である。
AI時代に評価されるのは「100%動く前提のコード」を書く人ではなく、「99.9%しか動かない前提でユーザー影響を最小化する構成」を設計できる人。
SLOを定義し、冗長化・フェイルオーバー・縮退運転をどう組むかが、システムの「壊れにくさ」を決める。

## 仕組みの要点

- **SLI（指標）/ SLO（目標）/ SLA（契約）** の3階層で考える
  - SLI: 実測値（例: 成功率、p99レイテンシ）
  - SLO: 内部目標（例: 月間成功率 99.9%）
  - SLA: 顧客との契約（SLOより緩く設定）
- **エラーバジェット**: 100% - SLO の余白。新機能リリース判断の根拠にする
- **可用性の数字感覚**: 99.9% = 月43分停止 / 99.99% = 月4.3分停止
- **冗長化の単位**: インスタンス / AZ / リージョン / クラウドプロバイダ。コストは指数的に増える
- **フェイルオーバー**: アクティブ-スタンバイ / アクティブ-アクティブ。RTO（復旧時間）/ RPO（データ損失許容）で設計
- **グレースフルデグラデーション**: 全断より「読み取り専用」「機能限定」で生かす

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| 「99.99%目指す」と無計画に宣言 | SLOを定量化し、エラーバジェットで運用判断 |
| 単一インスタンス + 「落ちたら再起動」 | min instances ≥ 2、ヘルスチェックで自動復旧 |
| DB障害でアプリ全停止 | キャッシュフォールバック + 読み取り専用モード |
| 全機能を同じSLOで扱う | クリティカルパス（決済等）と非クリティカル（推薦）で分離 |
| フェイルオーバーを設計したが訓練していない | 定期的にカオステスト・DR訓練を実施 |

## 設計例（FastAPI + Neon + Cloud Run）

```python
# 縮退運転: DB障害時に読み取り専用キャッシュで応答
from fastapi import FastAPI, HTTPException
import redis.asyncio as redis

app = FastAPI()
cache = redis.from_url("redis://...")

@app.get("/items/{id}")
async def get_item(id: str):
    try:
        return await db.fetch_item(id)  # Neon
    except DatabaseError:
        cached = await cache.get(f"item:{id}")
        if cached:
            return {"data": cached, "degraded": True}
        raise HTTPException(503, "service degraded")
```

```yaml
# Cloud Run: 最低2インスタンス + ヘルスチェック
service: api
spec:
  template:
    spec:
      containerConcurrency: 80
      containers:
        - image: gcr.io/.../api
          startupProbe: { httpGet: { path: /healthz } }
          livenessProbe: { httpGet: { path: /healthz } }
  traffic: [{ revisionName: api-v2, percent: 100 }]
  scaling: { minInstances: 2, maxInstances: 50 }
```

## トレードオフ

| 観点 | 高可用性側 | 低コスト側 |
|---|---|---|
| インスタンス数 | min 3+ AZ分散 | min 0（コールドスタート許容） |
| DB | マルチリージョンRead Replica | シングルリージョン |
| RTO | 数秒（アクティブ-アクティブ） | 数分（手動切替） |
| 月額コスト | 数倍〜10倍 | 最小 |
| 運用負荷 | 訓練・監視必要 | シンプル |

**判断軸**: ユーザー影響度 × 停止時間コスト > 冗長化コスト なら投資する。

## チェックリスト

- [ ] サービスごとにSLI/SLOを定量化し、ダッシュボード化している
- [ ] エラーバジェットがリリース判断に使われている
- [ ] クリティカルパスは min instances ≥ 2、AZ分散している
- [ ] DB/外部API障害時の縮退動作（キャッシュ・読み取り専用）が定義されている
- [ ] フェイルオーバー手順を年1回以上訓練している
