# トランザクションとACID特性

## 概要

トランザクションは「一連のDB操作をひとまとまりにする」仕組みで、ACID特性がその信頼性を保証する。
FastAPI + Neon（PostgreSQL）構成では、APIエンドポイントごとに暗黙的または明示的にトランザクションが発生する。
ACID を理解することで、「なぜロールバックが安全か」「なぜ長いトランザクションが危険か」を根拠を持って判断できる。

---

## 仕組みの要点

### ACID の4特性

| 特性 | 意味 | DBが何をしているか |
|---|---|---|
| **Atomicity（原子性）** | 全部成功か全部失敗か | WAL（ログ）でロールバック可能にする |
| **Consistency（一貫性）** | 制約・ルールを常に満たす | 外部キー・NOT NULL等の制約検証 |
| **Isolation（分離性）** | 並行トランザクション間の干渉を防ぐ | ロック or MVCC で実現 |
| **Durability（永続性）** | コミット後はクラッシュしても残る | WAL をディスクに flush してからコミット応答 |

### トランザクションの流れ

```
BEGIN
  → 操作1（INSERT / UPDATE / DELETE）
  → 操作2
  → 操作N
COMMIT（→ WAL が永続化されてデータ確定）
  or
ROLLBACK（→ 変更を全て取り消し）
```

### Isolation レベルとその問題

低い分離レベルは性能が上がるが、読み取り異常が起きる：

- **READ UNCOMMITTED**：コミット前の変更を読む（Dirty Read）※PostgreSQLでは実質RC扱い
- **READ COMMITTED**（PostgreSQL デフォルト）：コミット済みのみ読む。Non-repeatable Read は起きる
- **REPEATABLE READ**：同一トランザクション内で同じ行は同じ値。Phantom Read はまれに発生
- **SERIALIZABLE**：完全直列化。最も安全だが性能コストが高い

### PostgreSQL の MVCC（楽観的並行制御）

- 行の更新時に「新版」を作り、「旧版」は残す
- 読み取りはロックせず古いスナップショットを見る → 読み書きが干渉しない
- VACUUM が旧版（デッドタプル）を定期回収する

---

## パフォーマンス特性

- **短いトランザクション**：ロック保持時間が短く、並行性が高い
- **長いトランザクション**：MVCCの旧版が積み上がり、VACUUM が追いつかずテーブルが膨張
- **SERIALIZABLE**：SSI（Serializable Snapshot Isolation）を使うため、PostgreSQLでも比較的安価だが競合が増えるとリトライ発生

---

## コード例（FastAPI + asyncpg）

```python
import asyncpg

async def transfer(pool, from_id, to_id, amount):
    async with pool.acquire() as conn:
        async with conn.transaction():          # BEGIN / COMMIT 自動管理
            bal = await conn.fetchval(
                "SELECT balance FROM accounts WHERE id=$1 FOR UPDATE", from_id
            )
            if bal < amount:
                raise ValueError("残高不足")   # ここで例外 → 自動ロールバック
            await conn.execute(
                "UPDATE accounts SET balance=balance-$1 WHERE id=$2", amount, from_id
            )
            await conn.execute(
                "UPDATE accounts SET balance=balance+$1 WHERE id=$2", amount, to_id
            )
    # ブロックを抜けたら COMMIT
```

`FOR UPDATE` で行ロックを取り、残高チェックと更新の間に割り込みを防ぐ。

---

## よくある誤解・落とし穴

- **「READ COMMITTED で十分」は危険な場合がある**  
  残高チェック → 更新の間に別トランザクションが割り込める。`FOR UPDATE` か REPEATABLE READ が必要。

- **トランザクション外で例外を握りつぶすとデータが中途半端に残る**  
  例外をキャッチして何もしないと INSERT だけ成功した状態になり得る。

- **Neon のサーバーレスブランチはトランザクションをまたいでコネクションが切れる**  
  HTTP接続モードでは1リクエスト1トランザクションが基本。長いトランザクションはコネクション設計を確認すること。

- **VACUUM 不足でテーブル膨張**  
  長いトランザクションが走り続けると autovacuum が旧版を回収できず、クエリが遅くなる。

---

## 実務での使いどころ（FastAPI + Neon + Cloud Run）

- **FastAPI の依存注入でトランザクション境界を統一**  
  `get_db()` 依存でセッションを管理し、エンドポイント単位でコミット/ロールバックを自動化する。

- **決済・在庫更新などの多テーブル操作**  
  複数 UPDATE を1トランザクションに束ねることで、部分失敗を防ぐ。

- **Neon のブランチ機能（プレビュー環境）**  
  スキーママイグレーションをブランチ上で試してから main に適用。本番 WAL への影響ゼロ。

- **REPEATABLE READ を選ぶタイミング**  
  集計バッチや複数行の整合性が必要な読み取り処理。READ COMMITTED だと集計途中に他の更新が入り、集計がずれる。
