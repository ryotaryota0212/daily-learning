# トランザクションとACID特性

## 概要

トランザクションは「一連のDB操作をひとまとまりとして扱う仕組み」。全部成功するか全部失敗するかの二択を保証する。  
ACID特性はその保証の名前で、銀行振込・在庫更新・決済など「中途半端な状態が許されない処理」すべての土台になる。  
FastAPI + Neon（PostgreSQL互換）では async with session.begin() 一行がこの仕組みに乗っている。

---

## 仕組みの要点

### ACID の各特性

| 特性 | 意味 | 担保する障害 |
|------|------|-------------|
| **Atomicity（原子性）** | 全操作が成功 or 全ロールバック | 処理途中のクラッシュ |
| **Consistency（一貫性）** | 制約・外部キー・整合性が常に維持 | 不正データの混入 |
| **Isolation（分離性）** | 並行トランザクションが互いに干渉しない | 並行アクセスによる競合 |
| **Durability（耐久性）** | COMMIT後はクラッシュしてもデータが残る | ストレージ障害 |

### どうやって実現しているか

- **Atomicity** → WAL（Write-Ahead Log）にロールバック情報を書いてからデータを変更。クラッシュ時はログから元に戻す
- **Isolation** → ロック or MVCC（Multi-Version Concurrency Control）で並行制御
- **Durability** → COMMITはWALがディスクに到達した後に返す（fsync）

### 分離レベルと読み取り異常

分離性を「どこまで保証するか」でトレードオフがある：

| 分離レベル | ダーティ読み | ファジー読み | ファントム読み | PostgreSQLデフォルト |
|-----------|------------|------------|--------------|---------------------|
| READ UNCOMMITTED | 起こる | 起こる | 起こる | （実質RC扱い） |
| READ COMMITTED | 防ぐ | 起こる | 起こる | **デフォルト** |
| REPEATABLE READ | 防ぐ | 防ぐ | 起こる | |
| SERIALIZABLE | 防ぐ | 防ぐ | 防ぐ | 最強・最遅 |

- **ダーティ読み**：他トランザクションの未COMMITデータを読む
- **ファジー読み**：同一TX内で同じ行を2回読んだら値が変わっていた
- **ファントム読み**：同一TX内でSELECTしたら行数が変わっていた

---

## パフォーマンス特性

- 分離レベルが高い → 競合検出のオーバーヘッドが増える
- 長いトランザクションはロック保持時間が長くなり他のTXをブロック
- PostgreSQLのMVCCはロック競合を減らすが、**デッドタプル**が蓄積 → autovacuumが必要

---

## コード例（FastAPI + SQLAlchemy async）

```python
from sqlalchemy.ext.asyncio import AsyncSession

async def transfer(session: AsyncSession, from_id: int, to_id: int, amount: int):
    async with session.begin():          # BEGIN / COMMIT / ROLLBACK を自動管理
        sender = await session.get(Account, from_id, with_for_update=True)
        receiver = await session.get(Account, to_id, with_for_update=True)

        if sender.balance < amount:
            raise ValueError("残高不足")  # 例外でROLLBACKされる

        sender.balance -= amount
        receiver.balance += amount
        # session.begin() ブロックを抜ける時に自動COMMIT
```

- `with_for_update=True` → `SELECT ... FOR UPDATE`。同行への並行更新をブロック
- 例外が飛ぶと `session.begin()` が自動ロールバック

---

## よくある誤解・落とし穴

- **「autocommitはトランザクションなし」は誤り** → 各SQL文が単独のトランザクションになるだけ
- **READ COMMITTEDは十分安全とは限らない** → ファジー読み・ファントム読みが起きるユースケースでは分離レベルを上げる
- **長いトランザクションは避ける** → ロック保持 + MVCCのデッドタプル蓄積でパフォーマンス劣化
- **Neon（serverless）はコネクション切断でROLLBACK** → COMMITを確認するまで成功を信じない
- **SERIALIZABLEは遅いとは限らない** → 競合が少ない読み取り主体の処理ならREPEATABLE READと大差ない

---

## 実務での使いどころ（FastAPI + Neon + Cloud Run）

- **FastAPI**: `async with session.begin()` をルートハンドラ内に置く。例外ハンドラでロールバックを漏れなく行う
- **Neon**: コネクションプーリング（PgBouncer）使用時はトランザクションモードが推奨。セッションモードではサーバーレスコールドスタート中にコネクションが切れることがある
- **Cloud Run**: リクエストが途中でタイムアウトしてもDBは自動ロールバック → 冪等な処理設計と組み合わせると安全
- **在庫・決済処理**: `SELECT ... FOR UPDATE` でペシミスティックロック、または`UPDATE ... WHERE balance >= amount`でアトミックに更新してROWCOUNTを確認するパターンが実用的
