# 障害設計 — 何が壊れたらどうなるか・障害の連鎖を防ぐ・サーキットブレーカー

## 概要

システムは「壊れないか」ではなく「壊れたときにどう振る舞うか」で評価される。
障害設計とは、依存先の障害が自分のシステム全体を道連れにしないよう、
**連鎖障害（Cascading Failure）を構造的に防ぐ**技術の総称。
FastAPI + Cloud Run 構成では特に外部 API・DB・キャッシュへの依存点が障害起点になりやすい。

---

## 仕組みの要点

### 障害の種類と伝播パターン

- **タイムアウト障害**: 応答が返らずスレッド/コネクションを使い切る
- **連鎖障害**: A→B→C の依存で B が詰まると A も詰まり全体が落ちる
- **メモリリーク型**: 再試行ループがリソースを食い潰す
- **部分障害**: DB は生きているが特定クエリだけ遅い（検出が難しい）

### 主要な防御パターン

| パターン | 目的 | 概要 |
|---|---|---|
| タイムアウト | 無限待ちの防止 | 全外部呼び出しに上限を設定 |
| リトライ + Jitter | 一時障害の吸収 | 指数バックオフ＋ランダムずらし |
| サーキットブレーカー | 連鎖障害の遮断 | 失敗が閾値超えで呼び出しを止める |
| Bulkhead（隔壁） | リソース枯渇の局所化 | 機能ごとにスレッド/接続を分離 |
| Graceful Degradation | 部分機能提供 | コア機能は維持、非必須機能を落とす |

### サーキットブレーカーの状態遷移

```
Closed（正常）
  → 失敗率が閾値超え → Open（遮断中）
    → 一定時間後 → Half-Open（試験中）
      → 成功 → Closed / 失敗 → Open
```

---

## アンチパターン vs 正しい設計

| アンチパターン | 正しい設計 |
|---|---|
| タイムアウト未設定（デフォルト無限） | 全 HTTP クライアントに timeout=5s 等を明示 |
| 失敗したら即リトライを3回繰り返す | 指数バックオフ + Jitter（0.5〜1.5 倍のランダム） |
| 全機能が同一 DB 接続プールを共有 | 重要度別にプールを分ける（Bulkhead） |
| 障害時に 500 を返すだけ | キャッシュ or デフォルト値で Degraded 応答を返す |
| アラートがない → 気づかない | SLO ベースのアラート＋エラーバジェット管理 |

---

## コード例（FastAPI + httpx サーキットブレーカー）

```python
import httpx
from tenacity import retry, stop_after_attempt, wait_exponential, wait_random

# タイムアウト＋リトライ（Jitter 付き）
@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(min=1, max=8) + wait_random(0, 1),
    reraise=True,
)
async def call_external_api(url: str) -> dict:
    async with httpx.AsyncClient(timeout=5.0) as client:
        resp = await client.get(url)
        resp.raise_for_status()
        return resp.json()

# Graceful Degradation: 失敗時はキャッシュ値を返す
async def get_recommendations(user_id: str) -> list:
    try:
        return await call_external_api(f"/recommend/{user_id}")
    except Exception:
        return await get_cached_recommendations(user_id) or []
```

---

## トレードオフ

| 手法 | メリット | デメリット |
|---|---|---|
| タイムアウト短縮 | 障害の早期検出 | 正常な遅延リクエストも失敗扱いに |
| リトライ増加 | 一時障害に強い | 障害中のサーバ負荷増大（Retry Storm） |
| サーキットブレーカー | 連鎖障害の防止 | 実装コスト・状態管理が必要 |
| Graceful Degradation | ユーザー影響を最小化 | 「古い/不完全なデータ」を返すリスク |

---

## チェックリスト

- [ ] 全外部 HTTP 呼び出しにタイムアウトが設定されているか
- [ ] リトライは指数バックオフ + Jitter になっているか
- [ ] DB/外部 API の障害時に 500 以外の応答（Degraded）を返せるか
- [ ] Cloud Run の concurrency と DB 接続プール上限が整合しているか
- [ ] 障害シナリオを想定した負荷試験（Chaos Engineering）を実施しているか
