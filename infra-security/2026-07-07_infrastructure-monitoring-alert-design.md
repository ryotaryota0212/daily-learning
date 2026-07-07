# インフラのモニタリング・アラート設計

## 概要

障害を「起きてから知る」のではなく「起きる前に察知する」か「即座に検知する」ことが運用品質を決める。
モニタリングは単なるログ収集ではなく、**何を観測し・何を閾値にし・誰に通知するか**という設計判断の集合体。
Cloud Run + FastAPI + Neon の構成では、計測ポイントが分散するため統合的な可観測性（Observability）の設計が重要。

---

## 仕組みの要点

### Observability の3本柱

| 柱 | 内容 | 主なツール |
|---|---|---|
| Metrics | 数値の時系列データ | Cloud Monitoring, Prometheus |
| Logs | イベントの記録 | Cloud Logging, structlog |
| Traces | リクエストの経路追跡 | Cloud Trace, OpenTelemetry |

### 何を計測するか（SLIの選定）

- **RED メソッド**（サービス視点）
  - Rate: リクエスト数/秒
  - Errors: エラー率
  - Duration: レイテンシ（p50/p95/p99）
- **USE メソッド**（リソース視点）
  - Utilization: CPU・メモリ使用率
  - Saturation: キューの詰まり具合
  - Errors: リソースエラー数

### アラート設計の原則

- **SLO ベースでアラートを設定**する（閾値は感覚ではなくエラーバジェット消費率から逆算）
- **ページャーに送るアラート**はユーザー影響があるものだけ
- **Slack 通知**は傾向把握用（即対応不要なもの）
- アラートには「何が起きているか」「どう対応するか」のRunbookリンクを付ける

---

## アンチパターン vs 正しい設計

### アンチパターン

- CPU 使用率 80% で即アラート → **疲弊**（高いだけで問題ない場合が多い）
- すべてのログを構造化せず出力 → **検索不能**
- エラーログだけ監視、レイテンシ無視 → **遅延劣化を見逃す**
- アラート通知先が全員 → **誰も見ない**

### 正しい設計

- アラートは「p99 レイテンシ > 2秒 が 5分継続」のように**持続条件**を付ける
- ログは構造化（JSON）で出力し、`severity` / `trace_id` / `user_id` を必ず含める
- エラーバジェット（SLO 99.9% → 月43分の許容ダウン）を軸に優先度を決める
- オンコール体制と Runbook を事前に整備する

---

## コード/設計例

### FastAPI + structlog による構造化ログ

```python
import structlog
from fastapi import FastAPI, Request
import time

logger = structlog.get_logger()
app = FastAPI()

@app.middleware("http")
async def logging_middleware(request: Request, call_next):
    start = time.time()
    response = await call_next(request)
    duration_ms = (time.time() - start) * 1000

    logger.info(
        "http_request",
        method=request.method,
        path=request.url.path,
        status_code=response.status_code,
        duration_ms=round(duration_ms, 2),
        trace_id=request.headers.get("X-Cloud-Trace-Context", ""),
    )
    return response
```

### Cloud Monitoring アラートポリシー（Terraform 例）

```hcl
resource "google_monitoring_alert_policy" "high_error_rate" {
  display_name = "Cloud Run Error Rate > 5%"
  combiner     = "OR"

  conditions {
    display_name = "Error rate condition"
    condition_threshold {
      filter          = "resource.type=\"cloud_run_revision\" AND metric.type=\"run.googleapis.com/request_count\""
      comparison      = "COMPARISON_GT"
      threshold_value = 0.05
      duration        = "300s"  # 5分持続
    }
  }

  notification_channels = [google_monitoring_notification_channel.slack.name]
}
```

---

## トレードオフ

| 観点 | 細かく監視 | 粗く監視 |
|---|---|---|
| 検知精度 | 高い | 低い |
| アラート疲弊 | 起きやすい | 起きにくい |
| コスト | ログ・メトリクス量↑ | 低い |
| 推奨 | p95/p99 レイテンシ・エラー率 | CPU・メモリ単体 |

**SLO ベースアラート**が最もバランスが良い。ユーザー体験に直結する指標（エラー率・レイテンシ）を軸に、エラーバジェット消費速度でアラートを発火させる方式（Burn Rate Alert）が Google SRE の推奨。

---

## チェックリスト

- [ ] ログは JSON 構造化済みで `trace_id` を含む
- [ ] アラートは p95/p99 レイテンシとエラー率を SLO から逆算して設定
- [ ] 全アラートに Runbook リンクが付いている
- [ ] オンコール担当者と通知チャネルが明確に分離されている
- [ ] 月次でエラーバジェット消費量を確認するレビュー体制がある
