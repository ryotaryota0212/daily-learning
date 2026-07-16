# B-Treeインデックスの仕組みと計算量

## 概要

B-TreeはPostgreSQLをはじめほぼすべてのRDBMSがデフォルトで使うインデックス構造。
「なぜINDEXを貼るとSELECTが速くなるのか」「なぜ書き込みが遅くなるのか」を理解することで、
クエリチューニングやテーブル設計の判断が根拠を持ってできるようになる。
Neon(PostgreSQL)を使う場合も、実行計画（EXPLAIN）の読み方や設計判断がここに直結する。

---

## 仕組みの要点

### B-Treeの基本構造

```
         [30 | 60]          ← ルートノード
        /    |    \
  [10|20] [40|50] [70|80]   ← 内部ノード
  /  |  \  ...
葉  葉  葉                  ← リーフノード（実データのポインタ）
```

- **ノード** = ディスク上の1ページ（PostgreSQLは8KB）
- 各ノードは複数のキーを保持し、キー数+1の子を持つ（次数=m）
- **リーフノード**にのみ実際の行ポインタ（heap tuple）がある
- リーフノードは双方向リンクリストで繋がっている → 範囲検索が効率的

### 検索の流れ（`WHERE id = 45`）

1. ルートノードをメモリに読み込む
2. `45 > 30` かつ `45 < 60` → 中間の子ノードへ
3. `45 > 40` → 右側の子へ
4. リーフノードで `45` を発見 → heap tupleのポインタ取得
5. テーブル本体（heap）からページを読む

### 範囲検索（`WHERE id BETWEEN 40 AND 65`）

1. B-Treeを降りて `40` のリーフを見つける
2. リーフのリンクリストを右方向にスキャン（`65` まで）
3. 配列のフルスキャンより格段に速い

### 挿入の流れ（オーバーフロー処理）

- リーフノードに空きがあれば挿入するだけ
- 空きがなければ **ノード分割（split）**：
  1. ノードを2つに分ける
  2. 中央キーを親ノードに昇格させる
  3. 親もあふれたら再帰的に分割 → 最悪ルートが分割され木が1段深くなる

---

## 計算量・パフォーマンス特性

| 操作 | 計算量 | 備考 |
|------|--------|------|
| 検索（完全一致） | O(log n) | 木の高さ分のページI/O |
| 検索（範囲） | O(log n + k) | k = 結果件数 |
| 挿入 | O(log n) | split発生時は定数倍のコスト |
| 削除 | O(log n) | アンダーフロー時にマージ or 再分配 |
| フルスキャン | O(n) | Seq Scanより遅い場合も |

**木の高さ**: 次数mのB-Treeでn件のデータを格納すると高さ ≈ log_m(n)。
PostgreSQLの実測では数百万行でも高さ3〜4程度。つまり3〜4回のI/Oで検索完了。

---

## コード例

```sql
-- インデックス作成
CREATE INDEX idx_users_email ON users(email);

-- 実行計画の確認（Index Scanが使われているか）
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM users WHERE email = 'foo@example.com';

-- 複合インデックス（左端のカラムから順に使われる）
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at);

-- この検索はインデックスを使える（左端一致）
SELECT * FROM orders WHERE user_id = 1 AND created_at > '2026-01-01';

-- これは使えない（左端をスキップ）
SELECT * FROM orders WHERE created_at > '2026-01-01';
```

---

## よくある誤解・落とし穴

- **「カラムにINDEXを貼れば必ず速くなる」** → 誤り。テーブルが小さい場合やヒット率が高い場合、Seq Scanの方が速い。クエリオプティマイザが自動的に判断する
- **「複合インデックスの順番は関係ない」** → 誤り。`(a, b)` インデックスは `WHERE a = ?` や `WHERE a = ? AND b = ?` には使えるが `WHERE b = ?` 単独では使えない
- **「書き込みコストは無視できる」** → INSEPTのたびにB-Treeを更新するため、インデックスが多いほど書き込みが重くなる。不要なインデックスは削除する
- **「インデックスが使われているはず」** → `LIKE '%keyword'` は前方一致でないためインデックス使用不可。`LIKE 'keyword%'` なら使える
- **NULL値** → PostgreSQLのB-TreeはNULLを末尾に格納する。`IS NULL` 検索にもインデックスは使える（他のDBと異なる）

---

## 実務での使いどころ（FastAPI + Neon + Cloud Run）

- **Neonの外部キーカラムには必ずINDEX** → `user_id`, `order_id` 等。JOIN時に効く
- **作成日時の範囲検索** → `created_at` にINDEXを貼り、ページネーションを `OFFSET` ではなく `WHERE created_at < $cursor` で実装するとO(log n)に保てる（カーソルページネーション）
- **`EXPLAIN ANALYZE` を習慣化** → `Seq Scan` が出たら要注意。`Index Scan` か `Index Only Scan` が理想。`Bitmap Index Scan` は中間的なケース
- **Cloud Run（コネクション数制限）** → PgBouncerやNeon側のコネクションプーリングを使う場合、長時間トランザクションを減らす設計がB-Tree分割コストとも相性が良い
- **部分インデックス** → `CREATE INDEX ON orders(user_id) WHERE status = 'active'` のように条件付きINDEXでサイズ削減と速度向上を両立できる
