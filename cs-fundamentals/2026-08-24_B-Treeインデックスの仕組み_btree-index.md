# B-Treeインデックスの仕組みと計算量

## 概要

B-Tree（Balanced Tree）はデータベースのインデックスに使われる最重要データ構造。PostgreSQL（Neon）もデフォルトインデックスとしてB-Treeを採用している。ディスクI/Oを最小化しながら対数時間での検索・挿入・削除を実現する設計になっており、「なぜSELECTが速くなるか」「どのクエリでインデックスが使われないか」を理解する土台になる。

---

## 仕組みの要点

### B-Treeの構造

```
Root: [30 | 70]
      /     |     \
[10|20] [40|50|60] [80|90]
```

- **ノード（ページ）**：複数のキーを持つ（ページサイズ = 8KB が一般的）
- **次数 t**：各ノードが持てる最小/最大キー数を決める
  - 最小キー数: t-1、最大キー数: 2t-1
- **葉ノード**：実データへのポインタ（または実データ）を持つ
- **内部ノード**：子ノードへのポインタとキーを持つ

### 二分探索木（BST）との違い

| 特性 | BST | B-Tree |
|------|-----|--------|
| 1ノードのキー数 | 1 | 多数 |
| 木の高さ | 最悪O(n) | 常にO(log n) |
| ディスクI/O | 高い | 最小化 |
| バランス保証 | なし | 常に保たれる |

### 検索の流れ

1. ルートノードを読み込む（ディスクI/O: 1回）
2. キーと比較してどの子へ進むか決定
3. 葉ノードに到達するまで繰り返す
4. 高さ = log_t(n) なので I/O 回数は log_t(n) 回

### 挿入の流れ

1. 挿入先の葉ノードを探す
2. ノードに空きがあれば挿入（終了）
3. ノードが満杯（2t-1キー）なら**分割（Split）**
   - 中央のキーを親へ昇格
   - 左右2つの新ノードに分ける
4. 親も満杯なら再帰的に分割（最悪ルートまで伝播）

---

## 計算量

| 操作 | 計算量 |
|------|--------|
| 検索 | O(log n) |
| 挿入 | O(log n) |
| 削除 | O(log n) |
| 範囲検索 | O(log n + k)　k=結果件数 |

**なぜ log n か**：高さが常に ⌈log_t(n)⌉ に保たれるため。t が大きいほど木は低くなり I/O 回数が減る。

---

## コード例

```python
# B-Treeの効果をPostgreSQL(Neon)で確認する例

# インデックスなし → フルスキャン: O(n)
# インデックスあり → B-Tree検索: O(log n)

# EXPLAIN ANALYZEで実行計画を確認
query_without_index = """
EXPLAIN ANALYZE
SELECT * FROM orders WHERE customer_id = 12345;
"""
# → Seq Scan（全件スキャン）

query_after_index = """
CREATE INDEX idx_orders_customer_id ON orders(customer_id);

EXPLAIN ANALYZE
SELECT * FROM orders WHERE customer_id = 12345;
"""
# → Index Scan using idx_orders_customer_id
```

---

## よくある誤解・落とし穴

- **インデックスは常に速い**: レコード数が少ない場合はフルスキャンのほうが速い。オプティマイザが自動選択する
- **複合インデックスは順序が重要**: `(a, b, c)` のインデックスは `WHERE a=1 AND b=2` には使えるが `WHERE b=2` だけでは使えない（左端から使う）
- **関数適用でインデックス無効化**: `WHERE LOWER(email) = 'foo'` はインデックスを使えない。`WHERE email = LOWER('FOO')` にするか関数インデックスを作成する
- **更新コスト**: インデックスが増えるほど INSERT/UPDATE/DELETE が遅くなる。読み取り多用テーブルに絞って付ける
- **NULLの扱い**: PostgreSQLのB-TreeはNULLをインデックス化する（`IS NULL` でもインデックスが使える）

---

## 実務での使いどころ（FastAPI + Neon + Cloud Runスタック）

**Neon（PostgreSQL）でのインデックス設計**:

- `WHERE` 句で頻繁に使うカラム、`JOIN` の結合キー、`ORDER BY` のカラムに付ける
- `EXPLAIN ANALYZE` を本番に近いデータ量で実行して確認する（Neonは無料ティアでも実行可能）

**Cloud Runでの注意点**:
- コールドスタート時にコネクション数が急増 → コネクションプールが枯渇するとインデックスの速さが活きない
- Neonのサーバーレスドライバ（`@neondatabase/serverless`）やSQLAlchemy接続プールを適切に設定する

**FastAPIでのN+1問題との関係**:
- ORM（SQLAlchemy）でリレーションを `lazy load` にしていると N+1 クエリが発生
- インデックスを貼っても N+1 はインデックス効果を吹き飛ばす → `joinedload` や `selectinload` と組み合わせて使う

**典型的なインデックス追加パターン**:
```sql
-- 作成日時での範囲検索が多いなら
CREATE INDEX idx_posts_created_at ON posts(created_at DESC);

-- 複合条件が多いなら（カーディナリティ高い順に左へ）
CREATE INDEX idx_orders_status_user ON orders(user_id, status);
```
