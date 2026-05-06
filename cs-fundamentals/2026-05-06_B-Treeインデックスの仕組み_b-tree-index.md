# B-Treeインデックスの仕組みと計算量

## 概要

B-Tree（Balanced Tree）は、PostgreSQL・MySQLなどほぼすべてのRDBMSがデフォルトで使うインデックス構造。
「なぜINDEXを貼ると速くなるのか」「どういうクエリでは効かないのか」を理解することで、
スロークエリの原因特定やスキーマ設計の精度が上がる。
FastAPI + Neon（PostgreSQL）スタックでは、クエリのボトルネックの大半がインデックス設計に起因する。

---

## 仕組みの要点

### 木構造の基本

```
              [30 | 70]           ← ルートノード
             /    |    \
       [10|20]  [40|60]  [80|90]  ← 内部ノード
       /  |  \    ...              ← リーフノード（実データへのポインタ）
```

- **ノード** = ディスクの1ページ（PostgreSQLは8KB）
- **次数（order）m** = 1ノードが持てる最大キー数。ノードは常に ⌈m/2⌉ 以上のキーを保持（バランス維持）
- **リーフノード** = 実テーブルのタプル（行）へのポインタ（heap pointer）を保持
- **内部ノード** = 「このキー未満は左の子へ」という案内のみ

### 検索の流れ

1. ルートから比較を繰り返し、対象キーの葉まで降りる
2. 1ノードの比較はO(log m)だが、mは大きいので実質O(1)扱い
3. 木の高さ = log_m(N) → 全体でO(log N)

### 挿入・削除でバランスを保つ仕組み

- **挿入**: 葉に追加 → ノードが溢れたら**分割（split）**して親にキーを昇格
- **削除**: 削除後にキー数が不足したら**マージ（merge）**または隣から**借用（borrow）**
- どちらも木の高さを変えず、O(log N)を維持する

---

## 計算量・パフォーマンス特性

| 操作 | 計算量 | 備考 |
|------|--------|------|
| 点検索（=） | O(log N) | ルートから葉まで |
| 範囲検索（BETWEEN, >=） | O(log N + K) | Kは結果件数 |
| 挿入 | O(log N) | 分割が発生しても償却O(log N) |
| 削除 | O(log N) | マージが発生しても同上 |
| フルスキャン | O(N) | インデックスなしと同じ |

- **PostgreSQLの実測**: N=100万行でも木の高さは3〜4段程度（8KBページ、m≈数百）
- **ページキャッシュ**: 上位ノードはほぼ常にメモリに乗るため、実際のディスクI/Oは葉の1〜2回のみ

---

## コード例（クエリとINDEX設計）

```sql
-- 基本的なインデックス作成
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- 複合インデックス（左端から使われる）
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at);

-- 上記インデックスが使われるクエリ
SELECT * FROM orders
WHERE user_id = 42 AND created_at >= '2026-01-01';

-- 実行計画の確認（Neon/PostgreSQL）
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE user_id = 42;
-- "Index Scan using idx_orders_user_id" が出ればOK
-- "Seq Scan" が出たらインデックスが使われていない
```

---

## よくある誤解・落とし穴

- **`LIKE '%keyword%'` はインデックスが使えない**: 前方一致（`LIKE 'keyword%'`）はOK、中間/後方一致はNG（全文検索はGINインデックスを使う）
- **複合インデックスは左端から順に使われる**: `(user_id, created_at)`のインデックスは`WHERE created_at = ...`単体では使われない
- **低カーディナリティ列のインデックスは逆効果になりうる**: boolean型や区分コードなど値の種類が少ない列。プランナがSeq Scanを選ぶ場合がある
- **インデックスが多すぎると書き込みが遅くなる**: INSERT/UPDATE/DELETEのたびにすべてのインデックスを更新するため、書き込み主体のテーブルには貼りすぎ注意
- **NULL値はインデックスに含まれる（PostgreSQL）**: MySQLと挙動が異なる点に注意

---

## 実務での使いどころ（FastAPI + Neon + Cloud Run）

- **Neonのクエリ最適化**: `pg_stat_statements`や`EXPLAIN ANALYZE`でスロークエリを特定し、インデックスを追加
- **外部キー列には必ずインデックスを**: `orders.user_id`のようなJOIN/WHERE条件に使う列はマイグレーション時に忘れずに作成
- **部分インデックスで対象を絞る**: `CREATE INDEX ON orders(status) WHERE status = 'pending'` — 件数が少ない条件なら全件よりはるかに小さいインデックスになる
- **Cloud Runのコールドスタート対策**: コネクションプーリング（PgBouncer）と組み合わせると、インデックスの恩恵が接続オーバーヘッドに隠れにくくなる
- **マイグレーション時の注意**: 本番データが大量にあるとき `CREATE INDEX CONCURRENTLY` を使うことでテーブルロックを回避できる
