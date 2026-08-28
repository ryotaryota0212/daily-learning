# B-Treeインデックスの仕組みと計算量

## 概要

B-Tree（Balanced Tree）は、PostgreSQL・MySQL・SQLiteなど主要RDBMSのデフォルトインデックス構造。
検索・挿入・削除がすべてO(log n)で動作し、範囲検索にも対応する。
「なぜINDEXを張るとSELECTが速くなるか」「なぜカーディナリティが低い列にINDEXを張っても意味がないか」を理解するための基礎。

---

## 仕組みの要点

### 構造

- **ノード**：ディスクの1ページ（通常8KB）に対応
- **内部ノード**：キーと子ノードへのポインタを持つ
- **リーフノード**：キー＋実際の行データへのポインタ（ヒープタプルのctid）を持つ
- **次リーフへのポインタ**：リーフノード同士が連結リスト状に繋がる→範囲検索が高速

```
         [30 | 60]
        /    |     \
  [10|20] [40|50] [70|80]
    ↓↓      ↓↓       ↓↓
  (heap)  (heap)   (heap)
```

### 検索の流れ（例：WHERE id = 45）

1. ルートノードを読む → `45 > 30` かつ `45 < 60` → 中央の子へ
2. 内部ノード `[40|50]` を読む → `40 < 45 < 50` → 該当リーフへ
3. リーフノードでキーを見つけ、ctidからヒープを読む

### 範囲検索の流れ（例：WHERE id BETWEEN 40 AND 70）

1. `40` を持つリーフまでツリーを下りる
2. リーフの連結リストを右方向にたどりながら `70` まで収集
3. ソート済みなので追加ソート不要（`ORDER BY id` が無料になる場合もある）

### 挿入の流れ

1. 挿入先リーフを見つける
2. リーフに空きがあれば挿入
3. リーフが満杯 → **ページ分割（Split）** が起きる
   - リーフを2つに分割し、中央値を親に昇格
   - 最悪ケースではルートまで連鎖的に分割（高さが増える）

### ページ分割のコスト

- 書き込み時のレイテンシスパイクの主要因
- `FILLFACTOR`（デフォルト90%）で余白を確保し分割頻度を下げられる
- シーケンシャルINSERT（単調増加ID）は分割が末尾で起きるため効率的

---

## 計算量・パフォーマンス特性

| 操作         | 計算量     | 備考                              |
|------------|----------|-----------------------------------|
| 検索（点）    | O(log n) | ページI/Oの回数 ≈ ツリーの高さ      |
| 範囲検索     | O(log n + k) | k = 結果件数。リーフ連結リストをたどる |
| 挿入        | O(log n) | 分割が起きると定数倍のコスト          |
| 削除        | O(log n) | ページマージは遅延されることが多い    |

- ツリーの高さ ≈ log_B(n)（Bはページあたりのキー数 ≈ 数百）
- 1億行のテーブルでも高さ3〜4程度 → ディスクI/Oは数回で済む

---

## コード例（SQLレベル）

```sql
-- インデックス作成
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- 実行計画確認（EXPLAIN ANALYZE）
EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 42;
-- Index Scan using idx_orders_user_id on orders
--   Index Cond: (user_id = 42)
--   Rows Removed by Filter: 0

-- 複合インデックス（左端の列から順に使われる）
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at);

-- 範囲検索でも使える
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE user_id = 42 AND created_at > '2026-01-01';
```

---

## よくある誤解・落とし穴

- **低カーディナリティ列にINDEXを張っても逆効果**
  - `status IN ('active', 'inactive')` はフルスキャンの方が速い場合がある
  - PostgreSQLはコスト推定でINDEXを使わない選択をすることがある

- **複合インデックスは列順が重要**
  - `(user_id, created_at)` は `WHERE user_id = ?` には使えるが `WHERE created_at = ?` 単独では使えない

- **インデックスを張るほど書き込みが遅くなる**
  - INSERT/UPDATE/DELETEのたびにすべてのインデックスも更新される

- **NULLはインデックスに含まれない（PostgreSQLは含む）**
  - MySQLはNULLをインデックスに含まない → `WHERE col IS NULL` はフルスキャンになる

- **LIKE '%foo%' はインデックスを使えない**
  - 前方一致 `'foo%'` はINDEXを使える

---

## 実務での使いどころ（FastAPI + Neon + Cloud Run）

- **Neon（PostgreSQL）でのスロークエリ対策**
  - `EXPLAIN ANALYZE` で `Seq Scan` が出ていたらまずINDEXを疑う
  - `pg_stat_user_indexes` でINDEX使用率をモニタリング

- **FastAPIのAPIエンドポイント設計との対応**
  - フィルタリングに使う列（`user_id`、`status`、`created_at`）に複合インデックスを検討
  - ページネーション（`WHERE id > ?`）はシーケンシャルIDと相性が良い

- **Cloud RunのコールドスタートとDB接続**
  - コネクションプールが少ない場合、重いクエリがボトルネックになりやすい
  - 適切なINDEXでクエリを速くすることがコールドスタート対策にもなる

- **Neonのブランチ機能でINDEX検証**
  - 本番DBをブランチしてEXPLAIN ANALYZEで検証してからINDEX追加できる
