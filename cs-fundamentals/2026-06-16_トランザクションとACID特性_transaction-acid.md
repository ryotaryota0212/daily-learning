# トランザクションとACID特性

## 概要

トランザクションは「複数のDB操作をひとまとまりとして扱う仕組み」。  
ACID特性はトランザクションが保証すべき4つの性質を定義したもので、PostgreSQL（Neon含む）はこれをデフォルトで満たす。  
API設計やバグ調査で「なぜデータが中途半端な状態になるのか」を理解するための根幹知識。

---

## 仕組みの要点

### ACID の4特性

| 特性 | 意味 | 壊れると何が起きるか |
|------|------|----------------------|
| **A**tomicity（原子性） | 全操作が成功するか、全部ロールバックするか | 送金で「引き落とし済み・入金なし」が残る |
| **C**onsistency（一貫性） | トランザクション前後でDBの制約（FK/CHECK等）を満たす | 外部キー違反のデータが入る |
| **I**solation（分離性） | 並行トランザクションが互いに干渉しない | 読み書きが競合し不正な値を読む |
| **D**urability（永続性） | コミット済みデータはクラッシュ後も消えない | 障害後にコミット済みデータが失われる |

### 分離レベルと発生する問題

低い分離レベルほど性能は高いが、異常が起きやすい。

```
SERIALIZABLE          ← 最強保証、直列実行と等価
  ↑
REPEATABLE READ       ← PostgreSQLのデフォルト（実質MVCC）
  ↑
READ COMMITTED        ← 多くのDBのデフォルト
  ↑
READ UNCOMMITTED      ← PostgreSQLは未サポート（READ COMMITTEDにフォールバック）
```

| 分離レベル | Dirty Read | Non-Repeatable Read | Phantom Read |
|------------|-----------|---------------------|--------------|
| READ UNCOMMITTED | 起きる | 起きる | 起きる |
| READ COMMITTED | 防ぐ | 起きる | 起きる |
| REPEATABLE READ | 防ぐ | 防ぐ | 起きる※ |
| SERIALIZABLE | 防ぐ | 防ぐ | 防ぐ |

※ PostgreSQLのREPEATABLE READはMVCCによりPhantom Readも実質防ぐ

### 3つの代表的な異常

- **Dirty Read**: 未コミットのデータを他トランザクションが読む
- **Non-Repeatable Read**: 同じSELECTが同一TX内で異なる値を返す（他TXがUPDATEしてコミット）
- **Phantom Read**: 同じ範囲クエリの結果行数が変わる（他TXがINSERT/DELETE）

---

## PostgreSQLでのMVCC（Multi-Version Concurrency Control）

PostgreSQLはロックではなくMVCCで分離性を実現している。

- 各行に `xmin`（挿入したTXのID）と `xmax`（削除したTXのID）を持つ
- UPDATEは「旧行をxmaxで無効化 + 新行をxminで追加」
- 読み取りTXは自分のスナップショット時点で可視な行バージョンだけを見る
- 読み取りと書き込みが互いをブロックしない → 高並行性

---

## コード例

```python
# FastAPI + asyncpg でのトランザクション使用例
async def transfer(conn, from_id: int, to_id: int, amount: int):
    async with conn.transaction():
        balance = await conn.fetchval(
            "SELECT balance FROM accounts WHERE id=$1 FOR UPDATE", from_id
        )
        if balance < amount:
            raise ValueError("Insufficient funds")  # 自動ロールバック
        await conn.execute(
            "UPDATE accounts SET balance = balance - $1 WHERE id = $2", amount, from_id
        )
        await conn.execute(
            "UPDATE accounts SET balance = balance + $1 WHERE id = $2", amount, to_id
        )
        # ブロック終了時に自動COMMIT
```

`FOR UPDATE` で行レベルロックを取得し、競合する並行TXをブロックする。

---

## よくある誤解・落とし穴

- **「autocommitがデフォルト」**: psycopg2はデフォルト`autocommit=False`（暗黙TX）。asyncpg/SQLAlchemyは`autocommit=True`がデフォルト。ライブラリごとに確認必須
- **「SERIALIZABLEは遅い」**: PostgreSQL 9.1以降のSSI（Serializable Snapshot Isolation）はロックベースより高速。必要な場面では使う
- **「長いTXは問題ない」**: 長いTXはMVCCのゴミ行（dead tuples）を蓄積しautovacuumを妨害する。Neonでは特にxmin horizonが進まずストレージが膨れる
- **「ロールバックは無コスト」**: UNDOログの巻き戻しコストがある。大きなバルク操作のロールバックは時間がかかる

---

## 実務での使いどころ（FastAPI + Neon + Cloud Run）

- **FastAPI + asyncpg/SQLAlchemy**: `async with session.begin()` でTXスコープを明示。依存注入でセッション管理するのが定番
- **Neon（サーバーレスPostgreSQL）**: コールドスタート時のTX開始レイテンシに注意。コネクションプーリング（PgBouncer）をNeonが提供しているので活用する
- **Cloud Run（スケールアウト）**: 複数インスタンスが同時に同じ行を更新する競合が起きやすい。`SELECT FOR UPDATE` またはOCC（楽観的ロック）でハンドリングする
- **冪等性設計**: リトライ可能なAPIにするため、操作を冪等にする（`INSERT ... ON CONFLICT DO NOTHING` 等）とTXの再実行が安全になる
