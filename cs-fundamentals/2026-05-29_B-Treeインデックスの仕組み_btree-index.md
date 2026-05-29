# B-Treeインデックスの仕組みと検索・挿入の計算量

## 概要

B-Tree（Balanced Tree）はRDBMSが最も広く採用するインデックス構造。PostgreSQL（Neon）・MySQL・SQLiteのデフォルトインデックスはすべてB-Tree。
「なぜINDEXを張ると速くなるか」「どんなクエリには効かないか」を理解することで、スロークエリを自分で診断・修正できる力がつく。

---

## 仕組みの要点

### 木の構造

```
          [30 | 70]              ← ルートノード
         /    |    \
   [10|20] [40|50|60] [80|90]   ← 内部ノード
   /  |  \   ...       /  \
[葉] [葉] [葉]         [葉] [葉] ← 葉ノード（実データ or RowIDを保持）
```

- **ノード** = ディスクの1ページ（PostgreSQLは8KB）
- **葉ノード** は双方向リンクリストで連結 → 範囲スキャンが高速
- ルートから葉まで**常に同じ深さ**（Balanced）を保つ

### 検索の流れ（SELECT）

1. ルートノードをメモリに読む
2. キーを二分探索して次のノードへ
3. 葉ノードに到達 → ヒープ（テーブル本体）をRowIDで参照
4. 深さは `O(log_b N)`（bは1ノードに入るキー数、通常数百〜数千）

### 挿入の流れ（INSERT）

1. 葉ノードに空きがあれば挿入して完了
2. ノードが満杯 → **スプリット**（中央キーを親へ昇格、ノードを2分割）
3. スプリットが連鎖してルートまで伝播する場合あり（稀）

### インデックスの種類（PostgreSQL）

| 種類 | 用途 |
|------|------|
| B-Tree（デフォルト） | `=`, `<`, `>`, `BETWEEN`, `LIKE 'prefix%'` |
| Hash | `=` のみ、等値比較専用 |
| GIN | 配列・jsonb・全文検索 |
| BRIN | 挿入順に相関する大テーブル（ログ等） |

---

## 計算量・パフォーマンス特性

| 操作 | 計算量 |
|------|--------|
| 検索（点） | O(log N) |
| 範囲スキャン | O(log N + K)　K=取得件数 |
| 挿入・削除 | O(log N) |
| フルスキャン | O(N)　※インデックス不使用 |

- 実際の深さ：1億行でも高さ4〜5程度（ディスクI/O 4〜5回）
- **書き込み負荷**：挿入のたびにインデックスも更新 → インデックス過多は書き込みを遅くする

---

## コード例（PostgreSQL / SQLクエリと実行計画）

```sql
-- インデックス作成
CREATE INDEX idx_users_email ON users(email);

-- 複合インデックス（左端ルール: (a,b,c) → a, a+b, a+b+c の順で有効）
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at DESC);

-- 実行計画確認
EXPLAIN ANALYZE
  SELECT * FROM orders
  WHERE user_id = 42 AND created_at > '2026-01-01';
-- → Index Scan using idx_orders_user_created ... (cost=0.43..8.45 rows=10)

-- LIKE前方一致はB-Treeが効く
SELECT * FROM users WHERE email LIKE 'ryota%';  -- ✅ Index Scan
-- 後方一致・中間一致は効かない
SELECT * FROM users WHERE email LIKE '%gmail.com'; -- ❌ Seq Scan
```

---

## よくある誤解・落とし穴

- **「インデックスは多いほど良い」は誤り**。UPDATEのたびに全インデックスを更新するため、書き込み主体のテーブルでは逆効果
- **複合インデックスの左端ルール**を忘れがち。`(a, b)` インデックスは `WHERE b = ?` だけでは使われない
- **低カーディナリティ列**（性別・フラグ等）へのインデックスはプランナがSeq Scanを選ぶことがある（全行の数%を超えると行スキャンの方が速い）
- **NULL値**はB-Treeに含まれる（PostgreSQL）が、`IS NULL` 検索には効きにくい場面もある
- `EXPLAIN` と `EXPLAIN ANALYZE` の違い：前者は推定コスト、後者は実際に実行して実測値を出す

---

## 実務での使いどころ（FastAPI + Neon + Cloud Run）

- **Neon（PostgreSQL）**はサーバーレスでコールドスタートがある → インデックスを適切に張り、1クエリのI/O回数を最小化することが応答速度に直結
- FastAPIのエンドポイントで `user_id` や `created_at` を WHERE条件に使うなら、起動直後に `EXPLAIN ANALYZE` で実行計画を確認する習慣をつける
- **書き込みが多いAPIでは**不要なインデックスを削除してINSERT/UPDATE速度を守る
- `pg_stat_user_indexes` ビューで使われていないインデックスを定期的に棚卸しする

```sql
-- 未使用インデックスの確認
SELECT indexrelname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND indexrelname NOT LIKE 'pg_%';
```
