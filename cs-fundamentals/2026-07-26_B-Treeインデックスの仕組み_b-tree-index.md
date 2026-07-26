# B-Treeインデックスの仕組みと計算量

## 概要

B-TreeはPostgreSQLやMySQLが標準で使うインデックス構造。`SELECT`/`UPDATE`/`DELETE`のWHERE句を高速化するが、「なぜ速いか」を知らないと不要なインデックスを張ったり、使われないインデックスを作ったりしてしまう。NeonはPostgreSQLなので、B-Treeの挙動を理解することでクエリチューニングが直接できる。

## 仕組みの要点

### ツリー構造

```
              [30 | 70]              ← ルートノード
             /    |    \
      [10|20]  [40|60]  [80|90]     ← 内部ノード
      /  |  \    ...
[1-9][11-19][21-29]                 ← リーフノード（実データへのポインタ）
```

- **ノード = ディスクページ**（通常8KB）。1ノードに多数のキーを格納
- リーフノードは双方向リンクリストで連結 → 範囲検索が高速
- 全リーフが同じ深さ（Balanced）→ 最悪ケースでも同じ計算量

### 検索の流れ

1. ルートから開始、キーを比較して子ノードを選択
2. 深さ分だけディスクI/Oが発生（通常3〜4回）
3. リーフに到達 → テーブルのctid（行ポインタ）を取得 → Heap Fetchで実データ取得

### 挿入の流れ

1. 挿入位置のリーフを見つける
2. リーフに空きあり → そのまま挿入
3. リーフが満杯 → **ページ分割（Split）**: ノードを2つに分けて親へキーを昇格
4. 親も満杯なら再帰的に分割 → ルート分割でツリーの高さが増える

## 計算量・パフォーマンス特性

| 操作 | 計算量 | 備考 |
|------|--------|------|
| 検索 | O(log n) | nはレコード数、底はノードの分岐数（通常100以上） |
| 挿入 | O(log n) | ページ分割が発生する場合あり |
| 削除 | O(log n) | ページマージが発生する場合あり |
| 範囲検索 | O(log n + k) | kは結果件数、リーフの連結リストを辿る |

- 100万行でも深さ3〜4程度。I/Oが少ないのが強み
- インデックス自体もページにキャッシュされる → `shared_buffers`のヒット率が重要

## コード例（PostgreSQLでの確認）

```sql
-- インデックス作成と実行計画確認
CREATE INDEX idx_users_email ON users(email);

EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'test@example.com';
-- → Index Scan using idx_users_email（B-Tree使用）

-- 範囲検索もB-Treeは得意
EXPLAIN ANALYZE
SELECT * FROM orders WHERE created_at BETWEEN '2026-01-01' AND '2026-07-01';
-- → Index Scan / Bitmap Index Scan

-- インデックスサイズの確認
SELECT pg_size_pretty(pg_relation_size('idx_users_email'));
```

## よくある誤解・落とし穴

- **関数を使うとインデックスが効かない**  
  `WHERE LOWER(email) = 'foo'` はB-Treeを使えない。`CREATE INDEX ON users(LOWER(email))` で関数インデックスが必要

- **カーディナリティが低い列はB-Treeが非効率**  
  `is_active`（true/false）にB-Treeを張っても全体の半数取得には使われずSeqScanになる。Partial Indexを検討

- **複合インデックスの列順が重要**  
  `(a, b, c)` のインデックスは`WHERE a=1 AND b=2`には使えるが`WHERE b=2`だけには使えない（プレフィックス一致の法則）

- **INSERTが多いテーブルのインデックスは重い**  
  ページ分割が頻発するとWriteが遅くなる。不要なインデックスは張らない

- **`LIKE 'foo%'` は使えるが `LIKE '%foo'` は使えない**  
  前方一致はB-Treeで絞れるが後方・中間一致はSeqScan。pg_trgmのGINインデックスが代替

## 実務での使いどころ（FastAPI + Neon + Cloud Run）

- **Neon（PostgreSQL）はB-Treeがデフォルト**。`CREATE INDEX` で即座に有効
- FastAPIのエンドポイントで`WHERE user_id = :uid`が多いなら`user_id`に必ずインデックスを張る
- Cloud RunはコールドスタートでDBコネクションを張り直すため、接続後の最初のクエリが遅い。インデックスで個々のクエリ時間を短縮することが体験改善に直結
- `EXPLAIN ANALYZE` はNeonのコンソールまたはpsqlで実行できる。スロークエリを見つけたらまず実行計画を確認する
- マイグレーション時（Alembic等）に`CREATE INDEX CONCURRENTLY`を使うとロックなしでインデックスを作成できる（本番DBで重要）
