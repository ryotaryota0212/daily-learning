# トランザクションとACID特性

## 概要

トランザクションとは「一連の操作をひとまとまりとして扱う仕組み」。銀行振込のような「A口座から引き落とし → B口座へ入金」を途中で失敗させないために必要。ACID特性はDBがトランザクションに保証する4つの性質で、PostgreSQL（Neon）はこれをデフォルトで提供する。APIサーバーの障害やネットワーク断が起きても、データの整合性が壊れない土台になる。

---

## 仕組みの要点

### ACID の4特性

| 特性 | 意味 | 保証されないと何が起きるか |
|------|------|--------------------------|
| **Atomicity（原子性）** | 全部成功 or 全部ロールバック | 振込の引き落としだけ成功して入金されない |
| **Consistency（一貫性）** | 制約・ルールを常に満たす | 残高がマイナスになる |
| **Isolation（独立性）** | 並行トランザクションが互いに干渉しない | 読み途中のデータを別トランザクションが書き換える |
| **Durability（永続性）** | コミット後はクラッシュしても消えない | 再起動後にコミット済みデータが消える |

### Atomicityの実現方法

- DBは操作をすべて**WAL（Write-Ahead Log）**に書いてからデータファイルに反映
- クラッシュ時はWALを再生して「コミット済みのみ」を復元
- `ROLLBACK`を呼ぶとWALの未コミット分を無効化

### Isolationの分離レベル（重要）

```
低 ─────────────────────────────────────── 高
Read Uncommitted → Read Committed → Repeatable Read → Serializable
（汚読あり）     （PostgreSQLデフォルト）（幻読あり）    （完全直列化）
```

各レベルで起きる問題：

- **Dirty Read**：未コミットのデータを読む（Read Uncommittedのみ）
- **Non-repeatable Read**：同一行を2回読むと値が変わる（Read Committedで発生）
- **Phantom Read**：同一クエリで行数が変わる（Repeatable Readで発生）

---

## コード例（Python + psycopg2）

```python
import psycopg2

conn = psycopg2.connect(DATABASE_URL)
conn.autocommit = False  # デフォルトはFalse

try:
    with conn.cursor() as cur:
        cur.execute("UPDATE accounts SET balance = balance - 1000 WHERE id = 1")
        cur.execute("UPDATE accounts SET balance = balance + 1000 WHERE id = 2")
    conn.commit()
except Exception as e:
    conn.rollback()  # 両方の操作が取り消される
    raise
finally:
    conn.close()
```

SQLAlchemyでは `with session.begin():` のコンテキストマネージャが自動で commit/rollback する。

---

## 計算量・パフォーマンス特性

- 分離レベルを上げるほどロック競合が増えスループットが落ちる
- PostgreSQLはMVCC（Multi-Version Concurrency Control）で読み取りをロックフリーに実現
- `Serializable`はSSI（Serializable Snapshot Isolation）で実装されており、ロックではなく競合検出

---

## よくある誤解・落とし穴

- **「`autocommit = True`は速い」は誤解の場合がある**：個々のINSERTが毎回fsyncを呼ぶためバルク挿入は逆に遅い
- **長いトランザクションは危険**：ロックを長時間保持 → 他クエリをブロック。VACUUMも止まる（PostgreSQL）
- **`Read Committed`でも`SELECT`2回目の値は変わる**：ループ内で再読みする処理は注意
- **ネストしたトランザクションはSAVEPOINTで実現**：PostgreSQLに本物のネストはない

---

## 実務での使いどころ（FastAPI + Neon + Cloud Run）

- **FastAPI + SQLAlchemy**：`AsyncSession` + `async with session.begin()` でトランザクション管理。リクエストハンドラ単位でセッションを区切る
- **Neon（PostgreSQL）**：デフォルト分離レベルは`Read Committed`。金融・在庫系なら`Serializable`を明示指定する
- **Cloud Runの複数インスタンス**：並行リクエストが同じ行を更新するケースでは`SELECT ... FOR UPDATE`でロックを取る
- **バルク挿入の高速化**：`autocommit`モードで`COPY`コマンドを使うとトランザクションオーバーヘッドを削減できる
