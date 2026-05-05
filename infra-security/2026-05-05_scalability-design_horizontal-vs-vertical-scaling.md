# スケーラビリティ設計：水平スケール vs 垂直スケール、ボトルネック分析

## 概要

AI時代のシステムは「とりあえず動く」だけでは足りず、負荷増加に耐えられる設計が求められる。
スケーラビリティとは「負荷が増えたとき、どう対応できるか」の設計力。
正しい方向にスケールできるか否かは、アーキテクチャ選定の段階で決まる。
コードを書く前に「どこがボトルネックになるか」を予測できることが、設計者の最重要スキル。

---

## 仕組みの要点

### 垂直スケール（Scale Up）
- サーバー1台のCPU・メモリを増強する
- 実装変更なしに対応できる（手軽）
- 上限がある（物理限界・コスト壁）
- Cloud Run では `--cpu` / `--memory` を増やすだけ

### 水平スケール（Scale Out）
- サーバーを台数で増やす
- 理論上は無制限に拡張可能
- **ステートレス設計が前提**（セッションをサーバーに持たない）
- Cloud Run はデフォルトで水平スケール対応（リクエスト数でオートスケール）

### ボトルネックの3大パターン

| レイヤー | 症状 | 原因 |
|---------|------|------|
| アプリ (CPU) | レスポンス遅延、CPUスパイク | N+1クエリ、非効率ループ |
| DB | クエリ遅い、接続数枯渇 | インデックス不足、コネクションプール設計ミス |
| ネットワーク | タイムアウト、スループット低下 | 大きなペイロード、同期的な外部呼び出し |

---

## アンチパターン vs 正しい設計

### アンチパターン
- セッションをメモリ（インスタンス内）に保存 → 水平スケール不可
- DBへの直接接続をアプリで管理 → コネクション枯渇
- 全リクエストでDBフルスキャン → DB負荷が線形増加
- 「遅くなったらサーバーを増やせばいい」と思っている → DBはスケールしない

### 正しい設計
- セッション状態は Redis / Firestore 等の外部ストアへ
- DBはコネクションプーラー（PgBouncer / Neon の pooling mode）経由
- 頻繁に読まれるデータはキャッシュ（Cloud Memorystore / KV）
- 書き込み集中なら非同期キュー（Cloud Tasks / Pub/Sub）に逃がす

---

## コード/設計例

### Cloud Run + Neon のコネクションプール設定（FastAPI）

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

# Neon の pooling mode URL を使う（port 5432 → 6543）
DATABASE_URL = "postgresql+asyncpg://user:pass@host:6543/db?pgbouncer=true"

engine = create_async_engine(
    DATABASE_URL,
    pool_size=5,        # Cloud Run 1インスタンスあたりの接続数
    max_overflow=2,
    pool_pre_ping=True, # 死んだ接続を自動除去
)

AsyncSessionLocal = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)
```

### Cloud Run のスケール設定（cloudbuild.yaml 抜粋）

```yaml
- name: gcr.io/cloud-builders/gcloud
  args:
    - run
    - deploy
    - my-service
    - --min-instances=1    # コールドスタート防止
    - --max-instances=20   # コスト上限
    - --concurrency=80     # 1インスタンスあたりの同時リクエスト数
    - --cpu=1
    - --memory=512Mi
```

---

## トレードオフ

| 観点 | 垂直スケール | 水平スケール |
|------|------------|------------|
| 実装コスト | 低（設定変更のみ） | 中〜高（ステートレス設計が必要） |
| スケール上限 | 低（物理限界あり） | 高（理論上無制限） |
| 障害耐性 | 低（1点障害） | 高（1台落ちても継続） |
| コスト効率 | 小規模では有利 | 大規模で有利 |
| DBスケール | そのまま使える | DBがボトルネックになりやすい |

**原則：アプリ層は水平、DB層は垂直 + 読み取り分散（Read Replica）で補う**

---

## チェックリスト

- [ ] Cloud Run のコンカレンシー・インスタンス上限を意図的に設定しているか
- [ ] Neon / PostgreSQL に pooling mode（port 6543）で接続しているか
- [ ] セッション・一時状態をインスタンス内メモリに持っていないか
- [ ] 負荷増加時に最初にどのレイヤーが詰まるか予測できているか
- [ ] DBのスロークエリログを確認し、インデックス設計を検証したか
