# インフラのモニタリング・アラート設計

## 概要

「システムが壊れたことを誰より先に知る」のがモニタリングの目的。  
アラートが多すぎると無視され、少なすぎると障害を見逃す。  
設計の本質は「何が起きたか」ではなく「ユーザーへの影響を最速で検知し、根本原因を絞り込む」こと。  
AI時代でも、本番環境で何が起きているかを把握する力は代替不能なエンジニアリングスキル。

---

## 仕組みの要点

### 3層の観測シグナル（The Three Pillars）

| シグナル | 内容 | ツール例 |
|----------|------|----------|
| **Metrics** | 数値の時系列データ（CPU、レイテンシ、エラー率） | Cloud Monitoring, Prometheus |
| **Logs** | イベントの記録（構造化ログ推奨） | Cloud Logging, Loki |
| **Traces** | リクエストの処理経路（分散トレーシング） | Cloud Trace, Jaeger |

### SLI/SLO/SLAの関係

- **SLI**（Service Level Indicator）: 実測値。例: 過去5分のエラー率
- **SLO**（Service Level Objective）: 内部目標。例: エラー率 < 0.1%
- **SLA**（Service Level Agreement）: 契約上の保証。SLO より緩めに設定
- **エラーバジェット** = `1 - SLO`。消費ペースでアラートを設計する

### アラートの種類

- **症状ベース（Symptom-based）**: エラー率上昇、レイテンシ悪化 → ユーザー影響を直接検知
- **原因ベース（Cause-based）**: CPU 80%、ディスク残量 20% → 先手を打つ補助アラート
- 優先すべきは**症状ベース**。原因ベースは情報過多になりやすい

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---------------|-----------|
| CPU > 80% で即アラート | エラー率・P99レイテンシでSLO違反を検知 |
| アラートを全員に送る | オンコール担当に送り、エスカレーションルールを定義 |
| アラート条件が固定閾値 | エラーバジェット消費率でアラート（Burn Rate Alert） |
| ログを全部テキスト形式 | 構造化ログ（JSON）でフィルタリング・集計を可能にする |
| 全リクエストをトレース | サンプリング（1〜10%）でコスト最適化 |

---

## コード/設計例

### FastAPI + Cloud Run での構造化ログとメトリクス

```python
import logging, json, time
from fastapi import FastAPI, Request

logger = logging.getLogger()
app = FastAPI()

@app.middleware("http")
async def log_requests(request: Request, call_next):
    start = time.time()
    response = await call_next(request)
    duration_ms = (time.time() - start) * 1000
    # Cloud Loggingが severity を認識する構造化ログ
    logger.info(json.dumps({
        "severity": "ERROR" if response.status_code >= 500 else "INFO",
        "method": request.method,
        "path": request.url.path,
        "status": response.status_code,
        "duration_ms": round(duration_ms, 2),
    }))
    return response
```

### Burn Rate Alert の考え方

```
# SLO: 99.9% 可用性 = エラーバジェット 0.1% / 月 (約43分)
# Burn Rate = 実際のエラー率 / SLO 閾値
#
# Burn Rate 14x → 1時間でバジェットの14時間分を消費 → 即時ページング
# Burn Rate  6x → 6時間ウィンドウで検知 → チケット作成
#
# Cloud Monitoring でのアラート条件例:
#   (bad_requests / total_requests) > 0.014  # 14x burn rate
#   window: 1h, alignment_period: 5m
```

---

## トレードオフ

| 観点 | 多くモニタリング | 少なくモニタリング |
|------|-----------------|-------------------|
| 検知精度 | 高い | 見逃しリスク |
| アラート疲労 | 発生しやすい | 少ない |
| コスト | ログ・メトリクス費用増 | 低コスト |
| MTTR | 短縮しやすい | 原因特定に時間 |

**推奨バランス**: SLOの症状ベースアラートを最優先に整備し、原因ベースは補助として最小限に抑える。

---

## チェックリスト

- [ ] エラー率・レイテンシ（P50/P99）・可用性の SLI/SLO を定義している
- [ ] アラートは「症状ベース」を優先し、Burn Rate Alert を設定している
- [ ] ログは構造化（JSON）形式で severity・request_id・duration を含む
- [ ] オンコール担当とエスカレーションルールが明文化されている
- [ ] 月次でエラーバジェット消費率を振り返り、SLO を見直す運用がある
