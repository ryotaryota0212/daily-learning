# トランザクションとACID特性

## 概要

トランザクションは「複数の操作をひとまとまりとして扱う仕組み」。ACID特性はその信頼性を保証する4つの性質。
銀行振込・在庫更新・注文処理など、データの整合性が必要なあらゆる場面で基盤となる。
PostgreSQL（Neon）を使う時点で常にトランザクションの上で動いており、意図せず使っている場面も多い。
「なぜ一部の操作だけ失敗しないか」を理解することで、バグの少ないAPIが書ける。

---

## ACID特性の要点

### A — Atomicity（原子性）
- トランザクション内の操作は「全て成功」か「全て失敗（ロールバック）」のどちらか
- 途中でクラッシュしても中途半端な状態にならない
- 実現手段: **WAL（Write-Ahead Log）**。変更前にログを書き、障害時はログから巻き戻す

### C — Consistency（一貫性）
- トランザクション前後でDBの制約（FK、UNIQUE、NOT NULL）が常に満たされる
- 制約違反が起きた瞬間にロールバックされる
- アプリロジックの整合性まではDBは保証しない（それは開発者の責任）

### I — Isolation（独立性）
- 並行する複数のトランザクションが互いに干渉しないように見える
- 完全な独立は性能コストが高いため、**分離レベル**で調整する（後述）
- 実現手段: **MVCC（Multi-Version Concurrency Control）**。PostgreSQLは読み取りにロックをかけない

### D — Durability（永続性）
- コミット済みデータはクラッシュ後も消えない
- 実現手段: WALをディスクに書いてからコミット完了を返す

---

## トランザクション分離レベル

| レベル | ダーティリード | ノンリピータブルリード | ファントムリード |
|---|---|---|---|
| READ UNCOMMITTED | 起こる | 起こる | 起こる |
| READ COMMITTED | 防ぐ | 起こる | 起こる |
| REPEATABLE READ | 防ぐ | 防ぐ | 起こる |
| SERIALIZABLE | 防ぐ | 防ぐ | 防ぐ |

- PostgreSQL（Neon）のデフォルト: **READ COMMITTED**
- 用語:
  - ダーティリード: コミット前のデータを別TXが読む
  - ノンリピータブルリード: 同TX内で同じ行を2回読むと値が変わる
  - ファントムリード: 同TX内で同じ条件のSELECTが返す行数が変わる

---

## コード例（Python + SQLAlchemy + PostgreSQL）

```python
from sqlalchemy import text
from sqlalchemy.orm import Session

def transfer(db: Session, from_id: int, to_id: int, amount: int):
    # BEGIN は Session使用時に自動発行される
    sender = db.execute(
        text("SELECT balance FROM accounts WHERE id=:id FOR UPDATE"),
        {"id": from_id}
    ).fetchone()

    if sender.balance < amount:
        db.rollback()
        raise ValueError("残高不足")

    db.execute(text("UPDATE accounts SET balance=balance-:a WHERE id=:id"),
               {"a": amount, "id": from_id})
    db.execute(text("UPDATE accounts SET balance=balance+:a WHERE id=:id"),
               {"a": amount, "id": to_id})
    db.commit()  # ここで初めて永続化
```

- `FOR UPDATE`: 対象行に排他ロックをかけ、他TXの同時更新を防ぐ
- `db.rollback()`: 途中で失敗したら全操作を巻き戻す

---

## よくある誤解・落とし穴

- **「AutoCommitは安全」は誤り**: FastAPIのデフォルト設定によっては各SQL文が即コミットされ、ロールバックできない
- **「分離レベルを上げれば完全に安全」は誤り**: SERIALIZABLEはデッドロックのリスクが上がり、性能が落ちる
- **長いトランザクション**: セッションをまたいで長時間開いたままにするとロック競合が増える。FastAPIでは1リクエスト=1トランザクションを基本とする
- **MVCC下でもロックは存在する**: `SELECT FOR UPDATE`や`LOCK TABLE`は明示ロックをかける。読み取りのみMVCCでロックフリー

---

## 実務での使いどころ（FastAPI + Neon + Cloud Run）

- **FastAPI + SQLAlchemy**: `db.commit()` / `db.rollback()` を `try/except` で必ず対にする。`yield db` パターンでリクエスト終了時に自動クローズ
- **Neon（サーバーレスPostgreSQL）**: コネクションが短命なため長いトランザクションは避ける。`pgbouncer`（Neon内蔵）がコネクション再利用を担う
- **Cloud Run（スケールゼロ）**: コールドスタート時にコネクションが増えやすい。トランザクションは短く終わらせることでロック競合を防ぐ
- **べき等性の確保**: Cloud Runの再試行（retry）でAPIが2回呼ばれた場合でも、`ON CONFLICT DO NOTHING` や UUID主キーでデータ重複を防ぐ
