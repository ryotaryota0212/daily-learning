# トランザクションとACID特性

## 概要

トランザクションとは「一連のDB操作をひとまとまりとして扱う仕組み」。銀行振込のように「引き落とし成功・入金失敗」が起きると困る場面で不可欠。ACID特性はその信頼性を保証する4つの性質で、PostgreSQL（Neon）はすべてを満たす。APIで「べき等性」や「競合」を考えるとき、ACID理解が設計判断の土台になる。

---

## 仕組みの要点

### ACID の4特性

| 特性 | 意味 | 壊れると何が起きるか |
|------|------|------------------|
| **A**tomicity（原子性） | 全部成功 or 全部ロールバック | 「引き落としだけ成功」が起きる |
| **C**onsistency（一貫性） | 制約（FK・UNIQUE等）を常に満たす | 不正な状態がDBに入る |
| **I**solation（分離性） | 並行トランザクションが互いに干渉しない | 読み取り不整合・更新の上書き |
| **D**urability（永続性） | コミット済みデータはクラッシュ後も残る | 電源断でデータ消失 |

### Atomicity の実現

- 変更は最初 WAL（Write-Ahead Log）に書かれる
- コミット前にクラッシュ → ロールバック（WALの未コミット部分を無視）
- コミット後にクラッシュ → リカバリ（WALを再適用）

### Isolation レベル（重要）

```
低い ←————————————————→ 高い
READ UNCOMMITTED | READ COMMITTED | REPEATABLE READ | SERIALIZABLE
                 ↑PostgreSQLのデフォルト
```

- **Dirty Read**：コミット前の値を読む（RU のみ発生）
- **Non-Repeatable Read**：同一トランザクション内で同じ行を2回読むと値が変わる（RC で発生）
- **Phantom Read**：同一条件のSELECTで返る行数が変わる（RR以下で発生）
- **Serializable**：完全直列化。最も安全だがオーバーヘッド大

---

## パフォーマンス特性

- 分離レベルを上げるほどロック/MVCC のコストが増えスループット低下
- PostgreSQL はデフォルト `READ COMMITTED` → 多くのWebアプリに十分
- 長いトランザクションは**ロック競合・バキューム阻害**を引き起こすため短く保つ

---

## コード例（Python / SQLAlchemy + asyncpg）

```python
from sqlalchemy.ext.asyncio import AsyncSession

async def transfer(session: AsyncSession, from_id: int, to_id: int, amount: int):
    async with session.begin():          # トランザクション開始
        sender = await session.get(Account, from_id, with_for_update=True)
        receiver = await session.get(Account, to_id, with_for_update=True)

        if sender.balance < amount:
            raise ValueError("残高不足")  # 例外 → 自動ロールバック

        sender.balance -= amount
        receiver.balance += amount
    # begin() ブロック終了で自動コミット
```

- `with_for_update=True` → `SELECT ... FOR UPDATE` で行ロックを取得
- `session.begin()` のコンテキストマネージャが原子性を担保

---

## よくある誤解・落とし穴

- **「AutoCommitだから安全」は間違い**：1文ずつ即コミットされるため複数操作をまたぐ整合性は保てない
- **長いトランザクションに注意**：PostgreSQL は古いスナップショットを保持し続けるのでバキュームが走れなくなる（テーブル肥大化）
- **SERIALIZABLE ≠ 万能**：デッドロックやシリアライゼーション失敗が増え、リトライ設計が必要になる
- **READ COMMITTED でも競合は起きる**：「残高チェック → 更新」を別々のクエリで行うと TOCTOU（Time of Check to Time of Use）問題が発生。`FOR UPDATE` で防ぐ

---

## 実務での使いどころ（FastAPI + Neon + Cloud Run）

- **FastAPI**: `async with session.begin()` を1リクエスト1トランザクションの単位にする。失敗時は自動ロールバック
- **Neon（PostgreSQL）**: デフォルト `READ COMMITTED` のままでOK。競合が激しいエンドポイントのみ `REPEATABLE READ` を検討
- **Cloud Run（スケールアウト）**: 複数インスタンスが同じ行を更新する競合が起きやすい。`SELECT FOR UPDATE` または `UPDATE ... WHERE version = :v`（楽観的ロック）で対処
- **冪等API設計**: 同じリクエストが2回来ても安全にするため、`INSERT ... ON CONFLICT DO NOTHING` をトランザクション内で使う
