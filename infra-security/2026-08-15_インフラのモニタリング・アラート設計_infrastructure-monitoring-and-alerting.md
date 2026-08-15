# インフラのモニタリング・アラート設計

## 概要

監視は「何かが壊れたら知る」だけでなく、「壊れる前に検知し、障害の連鎖を防ぐ」ために存在する。
適切な監視設計がなければ、本番障害に気づくのがユーザーからの報告になる。
SLO駆動のアラート設計を理解することは「システムを成立させる力」の核心であり、
AI時代においてコードを書く以上に価値を持つスキルの一つ。

---

## 仕組みの要点

### 監視の4つの黄金シグナル（Google SRE）

- **レイテンシ**: リクエスト処理時間。成功と失敗を分けて計測する
- **トラフィック**: RPM/QPS。急増・急減の両方が異常シグナル
- **エラー率**: 5xx / 4xx の割合。SLOの基準になる
- **飽和度**: CPU/メモリ/DB接続数の使用率。限界に近づく前にアラート

### 監視の階層構造

```
ユーザー体験（外形監視: エンドポイントの死活確認）
  ↓ 遅い or 失敗している
アプリ層（エラー率、レイテンシ、成功率）
  ↓ どのサービスが原因か
インフラ層（CPU、メモリ、ディスク、ネットワーク I/O）
  ↓ DBやキャッシュは？
依存サービス層（Neon/PostgreSQL、外部API、Cloud Pub/Sub）
```

### メトリクス・ログ・トレースの3本柱

| 種別 | 役割 | ツール例 |
|---|---|---|
| メトリクス | 傾向把握・アラート | Cloud Monitoring, Prometheus |
| ログ | 原因特定・詳細調査 | Cloud Logging, structured JSON log |
| トレース | リクエスト連鎖の追跡 | Cloud Trace, OpenTelemetry |

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| CPU 80%超でアラート（原因監視） | エラー率・レイテンシ超過でアラート（ユーザー影響監視） |
| 全アラートを同じ優先度で発報 | P0（即時）/ P1（1h）/ P2（翌日）で分類 |
| アラートが多すぎて無視される | 行動可能なアラートのみ発報。Runbook必須 |
| ログだけで監視 | メトリクス + ログ + トレースの3本柱 |
| 手動で障害に気づく | SLOベースの自動アラートで先手を打つ |

---

## コード例: FastAPI + Cloud Run でのメトリクス計装

```python
# prometheus_client でメトリクスを公開する最小構成
from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST
from fastapi import Request, Response
import time

REQUEST_COUNT = Counter(
    "http_requests_total", "Total HTTP requests",
    ["method", "path", "status"]
)
REQUEST_LATENCY = Histogram(
    "http_request_duration_seconds", "Request latency",
    ["path"]
)

@app.middleware("http")
async def metrics_middleware(request: Request, call_next):
    start = time.time()
    response = await call_next(request)
    duration = time.time() - start
    REQUEST_LATENCY.labels(path=request.url.path).observe(duration)
    REQUEST_COUNT.labels(
        method=request.method,
        path=request.url.path,
        status=response.status_code
    ).inc()
    return response

@app.get("/metrics")
async def metrics():
    return Response(generate_latest(), media_type=CONTENT_TYPE_LATEST)
```

---

## SLOベースのアラート設計

```yaml
# エラーバジェットアラートの設定例（Google Cloud Monitoring 概念）
SLO: 99.9% 可用性（月間 43.8 分まで許容）

アラート条件:
  P0（即時対応）: 過去1時間のエラー率 > 1.0%  # バジェットが急速に消費中
  P1（1h以内）:   過去6時間のエラー率 > 0.5%  # バジェット消費が加速中
  P2（翌日確認）: 過去24時間でバジェットの50%を消費
```

**ポイント**: 症状（ユーザーへの影響）でアラートし、原因（CPUなど）は調査時に参照する。

---

## トレードオフ

- **感度 vs ノイズ**: アラート閾値を下げると誤検知が増え、アラート疲れ（alert fatigue）を引き起こす
- **リアルタイム vs コスト**: 高頻度なメトリクス収集はコスト増。重要度に応じて収集間隔を分ける
- **外形監視 vs 内部監視**: 外形監視はユーザー視点で実際の障害を捉えるが、原因特定が難しい
- **集中管理 vs 分散**: Cloud Monitoring 等の集中型は管理しやすいが単一障害点になる可能性
- **詳細ログ vs コスト**: ログ量が増えるとストレージ・クエリコストが上昇。重要度でフィルタリング

---

## チェックリスト

- [ ] 4つの黄金シグナル（レイテンシ・トラフィック・エラー率・飽和度）をすべて計測しているか
- [ ] SLOに基づいたアラート閾値を設定しているか（原因ベースではなく症状ベース）
- [ ] アラートに「誰が・何を・いつまでに」対応するかの Runbook が紐づいているか
- [ ] メトリクス・ログ・トレースの3本柱でインシデント時の原因追跡ができるか
- [ ] ダッシュボードでシステム全体のヘルスを一目で確認できるか
