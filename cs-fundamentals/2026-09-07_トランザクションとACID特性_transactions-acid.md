# トランザクションとACID特性

## 概要

トランザクションは「一連の操作をひとまとまりとして扱う仕組み」。銀行振込のように「引き落としと入金が必ず両方成功する（or 両方失敗する）」を保証する。ACID特性はその信頼性の4つの柱。Neon（PostgreSQL）を使うAPIでも、注文・決済・在庫更新など複数テーブルをまたぐ操作で毎回関係する。

---

## 仕組みの要点

### ACID の4特性

| 特性 | 意味 | 具体例 |
|------|------|--------|
| **Atomicity（原子性）** | 全操作が成功するか、全部ロールバックか | 振込失敗 → 引き落としも取り消し |
| **Consistency（一貫性）** | トランザクション前後でDB制約が満たされる | 残高がマイナスにならない |
| **Isolation（分離性）** | 並行トランザクションが互いに干渉しない | 別セッションが中途状態を読まない |
| **Durability（永続性）** | COMMITしたデータはクラッシュ後も残る | WALによりディスクに記録済み |

### トランザクションの基本フロー

```
BEGIN
  → 操作A（INSERT/UPDATE/DELETE）
  → 操作B
  → エラーなし？ → COMMIT（確定）
              ↘ エラーあり → ROLLBACK（取り消し）
```

### 分離レベル（Isolation Levels）と読み取り異常

| 分離レベル | ダーティリード | ノンリピータブルリード | ファントムリード |
|-----------|:-----------:|:-----------------:|:----------:|
| READ UNCOMMITTED | 発生 | 発生 | 発生 |
| READ COMMITTED   | なし | 発生 | 発生 |
| REPEATABLE READ  | なし | なし | 発生 |
| SERIALIZABLE     | なし | なし | なし |

- PostgreSQL のデフォルトは **READ COMMITTED**
- 高い分離 = ロック競合増加 = スループット低下のトレードオフ

---

## パフォーマンス特性

- トランザクションが長い → ロック保持時間が増える → 他の処理がブロック
- `COMMIT` のコストは WAL のフラッシュ（ディスク I/O）が支配的
- 小さいトランザクションを素早く閉じるのが基本戦略

---

## コード例（Python + asyncpg / psycopg）

```python
import asyncpg

async def transfer(pool, from_id: int, to_id: int, amount: int):
    async with pool.acquire() as conn:
        async with conn.transaction():          # BEGIN / COMMIT自動
            balance = await conn.fetchval(
                "SELECT balance FROM accounts WHERE id=$1 FOR UPDATE",
                from_id,
            )
            if balance < amount:
                raise ValueError("残高不足")   # → 自動ROLLBACK
            await conn.execute(
                "UPDATE accounts SET balance = balance - $1 WHERE id=$2",
                amount, from_id,
            )
            await conn.execute(
                "UPDATE accounts SET balance = balance + $1 WHERE id=$2",
                amount, to_id,
            )
            # ブロック終了時に COMMIT
```

---

## よくある誤解・落とし穴

- **「READ COMMITTED で十分」は誤り** — 在庫の二重引き落としなど「先読み→書き込み」パターンは REPEATABLE READ か `SELECT ... FOR UPDATE` が必要
- **トランザクション内でHTTP呼び出しはNG** — ロック保持中に外部I/Oが入ると長時間ブロックが発生
- **暗黙のトランザクション** — ORMは自動で BEGIN するので、明示的 COMMIT/ROLLBACK がなくてもトランザクションは動いている
- **Durability の誤解** — `fsync=off` や `synchronous_commit=off` にすると性能は上がるがクラッシュ時データ消失リスクあり（Neon はデフォルト安全）

---

## 実務での使いどころ（FastAPI + Neon + Cloud Run）

- **注文API** — 在庫デクリメント + 注文レコード挿入 + 支払いステータス更新を1トランザクションで囲む
- **非同期コード** — `asyncpg` の `conn.transaction()` コンテキストマネージャが最もシンプル
- **Cloud Run のスケールアウト** — 複数インスタンスが同時に書き込む → 分離レベルと `FOR UPDATE` の設計が競合バグを防ぐ
- **Neon のコネクションプーリング** — トランザクションモードプーリングでは「1トランザクション=1コネクション」なので、長いトランザクションはプールを枯渇させないよう注意
