# トランザクションとACID特性

## 概要

トランザクションは「一連のDB操作をひとまとまりとして扱う仕組み」。ACID特性はその正しさを保証する4つの性質。  
銀行振込・在庫更新・予約処理など、「途中で失敗したら全部なかったことにしたい」あらゆる場面で基盤になる。  
NeonはPostgreSQLベースのためACID準拠。FastAPIで複数テーブルを更新する際、トランザクション境界を意識しないとデータ不整合が起きる。

---

## ACID特性の要点

| 性質 | 意味 | 実装の仕組み |
|------|------|-------------|
| **Atomicity（原子性）** | 全部成功 or 全部失敗 | ROLLBACK / WAL |
| **Consistency（一貫性）** | 制約・外部キーを常に満たす | 制約チェック |
| **Isolation（分離性）** | 同時実行しても互いに干渉しない | ロック / MVCC |
| **Durability（永続性）** | COMMITしたら障害後も消えない | WAL / fsync |

### 各性質の具体像

- **Atomicity**: `BEGIN → 操作A → 操作B → COMMIT` の途中でクラッシュしても、次回起動時にロールバックされる
- **Consistency**: `NOT NULL` や `FOREIGN KEY` 制約はCOMMIT時に検証される（延期可能）
- **Isolation**: 同時に1000件のリクエストが来ても、各トランザクションは「自分だけが動いている」ように見える
- **Durability**: COMMITが返った時点でWAL（ジャーナル）に書き込み済み。ディスク障害でもリカバリ可能

---

## 分離レベル（Isolation Levelsの比較）

分離性には「どこまで厳密か」のグレードがある。厳密にするほど並行性が下がる。

| レベル | Dirty Read | Non-repeatable Read | Phantom Read |
|--------|-----------|---------------------|--------------|
| READ UNCOMMITTED | 起きる | 起きる | 起きる |
| READ COMMITTED | なし | 起きる | 起きる |
| REPEATABLE READ | なし | なし | 起きる |
| SERIALIZABLE | なし | なし | なし |

- PostgreSQL/Neonのデフォルトは **READ COMMITTED**
- 金融系など厳密さが必要なら **SERIALIZABLE**（ただしデッドロックリスク増）

---

## コード例

```python
# SQLAlchemy + FastAPI でのトランザクション管理
from sqlalchemy.orm import Session

def transfer(db: Session, from_id: int, to_id: int, amount: int):
    try:
        sender = db.query(Account).filter_by(id=from_id).with_for_update().first()
        receiver = db.query(Account).filter_by(id=to_id).with_for_update().first()

        if sender.balance < amount:
            raise ValueError("残高不足")

        sender.balance -= amount
        receiver.balance += amount
        db.commit()          # ここで初めて永続化
    except Exception:
        db.rollback()        # 失敗時は全部巻き戻し
        raise
```

- `with_for_update()` = SELECT FOR UPDATE（排他ロック）で並行更新を防ぐ
- `db.rollback()` を明示しないとセッションが汚染されたまま残る

---

## よくある誤解・落とし穴

- **「AutoCommitがデフォルト」と思い込む**: SQLAlchemyはデフォルトでトランザクション内。ただしFastAPIの依存注入パターンで `db.commit()` を忘れると変更が保存されない
- **長いトランザクションはNG**: トランザクションを開きっぱなしにするとロック保持時間が伸び、デッドロックや接続枯渇の原因になる
- **分離レベルを上げれば安心ではない**: SERIALIZABLEはデッドロック時にエラーを返す。リトライ処理が必須
- **READ COMMITTEDの落とし穴**: 同一トランザクション内で同じ行を2回読むと値が変わることがある（Non-repeatable Read）

---

## 実務での使いどころ（FastAPI + Neon + Cloud Run）

- **複数テーブル更新**: 注文テーブルと在庫テーブルを同時更新するときは必ず1トランザクションにまとめる
- **Neonのサーバーレス接続**: コネクションプールが短命なため、トランザクションを長く保持しない設計が重要
- **Cloud Runのスケールアウト**: 複数インスタンスが同時にDBを叩くため、`SELECT FOR UPDATE` や楽観的ロックでの競合制御が必要
- **リトライ設計**: SERIALIZABLEを使う場合は `sqlalchemy.exc.OperationalError` をキャッチしてリトライするミドルウェアを用意する
