# B-Treeインデックスの仕組みと計算量

## 概要

B-Tree（Balanced Tree）はPostgreSQLをはじめとほぼすべてのRDBMSがデフォルトで使うインデックス構造。
なぜこれが選ばれるのかを理解すると、「インデックスをどこに張るか」「なぜ効かないのか」を自分で判断できる。
FastAPI + Neonスタックでスロークエリを最適化するとき、EXPLAINの実行計画が読めるようになる。

---

## 仕組みの要点

### 木構造の特徴

```
          [30 | 70]                ← ルートノード
         /    |    \
   [10|20] [40|50|60] [80|90]    ← 内部ノード
   /  |  \   ...       ...
[行]  [行]  [行]                  ← リーフノード（実データへのポインタ）
```

- **次数（Order）m**: 各ノードが持てる最大キー数。PostgreSQLは通常1ページ=8KBに収まる数百個
- **バランス保証**: すべてのリーフが同じ深さ → 最悪ケースも O(log N)
- **リーフ間リンク**: B+Tree（PostgreSQLが採用）はリーフ同士が双方向連結リスト → 範囲検索が速い

### 検索の流れ

1. ルートから開始。キーと比較して左/右の子へ降りる
2. リーフに到達 → 行のTID（物理アドレス）を取得
3. ヒープファイルから実行の行を読む（Index Scanの場合）

### 挿入・分割

- リーフが満杯 → ノードを分割し、中央キーを親へ昇格
- 親も満杯なら再帰的に分割 → 最終的にルートが分割されて木が1段高くなる

---

## 計算量・パフォーマンス特性

| 操作         | 計算量      | 備考                          |
|------------|-----------|------------------------------|
| 検索（等値）   | O(log N)  | ディスクI/Oがボトルネック            |
| 検索（範囲）   | O(log N + K) | K=取得行数。リーフリンクを辿る      |
| 挿入         | O(log N)  | 分割が発生しても定数回               |
| 削除         | O(log N)  | 再バランス（マージ）あり             |
| フルスキャン   | O(N)      | インデックスなしと同じ               |

- **ディスク1回のI/O = 1ノード読み込み**。ページサイズが大きいほど浅い木になりI/O回数が減る
- テーブル100万行 → 木の深さ約3〜4段 → 検索に必要なI/O = 3〜4回

---

## コード例（EXPLAIN ANALYZEの読み方）

```sql
-- インデックスあり
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 42;
-- Index Scan using orders_user_id_idx on orders
--   Index Cond: (user_id = 42)
--   Rows Removed by Filter: 0
--   Buffers: shared hit=4  ← I/O 4回（木の深さ）

-- インデックスなし or 選択率が低いとSeq Scanになる
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'pending';
-- Seq Scan on orders  ← 全行スキャン
--   Filter: (status = 'pending')
--   Rows Removed by Filter: 980000
```

---

## よくある誤解・落とし穴

1. **左端一致の原則を忘れる**
   複合インデックス `(a, b, c)` は `WHERE a=1 AND b=2` には使えるが `WHERE b=2` だけでは使えない

2. **関数をかけるとインデックスが無効になる**
   `WHERE LOWER(email) = 'foo@example.com'` → `email` のインデックスは使われない
   → 関数インデックス `CREATE INDEX ON users (LOWER(email))` で回避

3. **低選択率カラムへのインデックスは逆効果になることがある**
   `status` が 'active'/'inactive' の2値 → ほぼ全行取得するならSeq Scanの方が速い
   → オプティマイザが自動判断するが、統計情報が古いと誤判断する

4. **NULLとインデックス**
   PostgreSQLはB-TreeにNULLを含める。`IS NULL` 検索もインデックスが使える

5. **書き込み性能のトレードオフ**
   インデックスは読み取りを速くするが、INSERT/UPDATE/DELETEのたびに更新コストがかかる

---

## 実務での使いどころ（FastAPI + Neon + Cloud Run）

- **Neon = PostgreSQL**: B-Treeがデフォルト。追加設定なしで有効
- **サーバーレスの接続コスト**: Neonはコールドスタート時にページキャッシュがない → インデックスの深さが浅いほど有利
- **複合インデックス設計**: `WHERE user_id = ? ORDER BY created_at DESC` が頻出なら `(user_id, created_at DESC)` の複合インデックスが有効（Index Scanのみで完結）
- **EXPLAIN ANALYZE習慣化**: FastAPIのエンドポイントで遅いクエリを見つけたら `Seq Scan` を `Index Scan` に変えることを最初に疑う
- **部分インデックス**: `CREATE INDEX ON orders (user_id) WHERE status = 'active'` → アクティブなレコードのみ対象にして小さく保つ
