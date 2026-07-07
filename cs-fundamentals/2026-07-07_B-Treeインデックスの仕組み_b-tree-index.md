# B-Treeインデックスの仕組みと計算量

## 概要

B-Tree（バランス木）はRDBMS（PostgreSQL・MySQLなど）が標準採用するインデックス構造。  
ディスクI/Oを最小化するよう設計されており、検索・挿入・削除すべてO(log n)を保証する。  
「なぜINDEXを貼ると速くなるか」「どんなクエリでは効かないか」を理解することで、クエリチューニングの判断力が上がる。  
NeonはPostgreSQLエンジンなので、この知識はそのままスロークエリ対策に使える。

---

## 仕組みの要点

### 木構造の構成

```
              [30 | 60]              ← ルートノード
             /    |    \
       [10|20]  [40|50]  [70|80]    ← 内部ノード
       /  |  \   ...        ...
[1..9][11..19]...                   ← リーフノード（実際のデータ行へのポインタ）
```

- **ノード** = ディスクの1ページ（PostgreSQLはデフォルト8KB）
- 1ノードに多数のキーを格納し、木の高さを低く抑える → I/O回数が少ない
- リーフノードは双方向リンクで連結 → 範囲検索が効率的

### 検索の流れ

1. ルートノードをメモリに読み込む（1回のI/O）
2. キーと比較して適切な子ノードへ降りる
3. リーフノードに到達 → 行ポインタ（TID）を取得
4. テーブルヒープから実際の行データを取得

### 挿入・分割

- リーフが満杯になると **ノード分割** が発生し、親にキーが押し上げられる
- 分割はO(log n)回以下で波及 → 木の高さは常にバランスが保たれる

### PostgreSQLでの実装詳細

- 実装は `src/backend/access/nbtree/`
- 高さは通常3〜4段。テーブルが億行規模でも4回のI/Oで到達
- リーフページにはMVCC用のデッドタプルが積まれるため、`VACUUM`でページを整理

---

## 計算量・パフォーマンス特性

| 操作 | 計算量 | 備考 |
|------|--------|------|
| 検索（等値） | O(log n) | 木の高さ分のI/O |
| 検索（範囲） | O(log n + k) | k=結果件数。リーフを横断 |
| 挿入 | O(log n) | 分割が起きても変わらない |
| 削除 | O(log n) | マーク削除 → VACUUMで物理削除 |
| フルスキャン | O(n) | インデックス不使用時 |

- **選択率が高い**（=結果が全体の10%超）場合はSeqScanの方が速いことが多い
- インデックスのサイズは概ねデータサイズの10〜30%

---

## コード例（PostgreSQL）

```sql
-- インデックス作成と実行計画確認
CREATE INDEX idx_orders_user_id ON orders(user_id);

EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 42;
-- Index Scan using idx_orders_user_id → O(log n)

-- 複合インデックスは左端から使われる
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at);

EXPLAIN ANALYZE
SELECT * FROM orders
WHERE user_id = 42 AND created_at >= '2026-01-01';
-- Index Scan （両方のカラムが使われる）

EXPLAIN ANALYZE
SELECT * FROM orders WHERE created_at >= '2026-01-01';
-- Seq Scan （user_idがないと複合インデックスは使えない）
```

---

## よくある誤解・落とし穴

- **「インデックスは多ければ多いほど良い」** → 誤り。INSERTのたびに全インデックスを更新するため書き込みが遅くなる
- **関数をかけるとインデックスが効かない** → `WHERE LOWER(email) = ...` はSeqScan。対策は関数インデックス `CREATE INDEX ON users(LOWER(email))`
- **複合インデックスの列順** → `(a, b)` のインデックスは `WHERE b = ...` だけでは使えない。等値条件のカラムを先にする
- **NULL値** → B-TreeはNULLをインデックスに含める（PostgreSQL）。`IS NULL` でもIndex Scanが効く
- **カーディナリティが低い列**（例: boolean）→ 全体の50%が引っかかる場合はSeqScanを選ばれる

---

## 実務での使いどころ（FastAPI + Neon + Cloud Runスタック）

- **Neon（PostgreSQL）のスロークエリ対策**: `EXPLAIN ANALYZE` でSeqScanが出たらまず複合インデックスを検討
- **FastAPIの一覧API**: ページネーション実装時に `ORDER BY id` のカラムにインデックスがあるか確認。Cursorベースページネーションはインデックスを活かしやすい
- **Cloud Runのコールドスタート**: コネクション確立後の最初のクエリが重い場合、インデックスがウォームアップされていないことが原因になることがある
- **Neonのブランチ機能**: テスト用ブランチで `CREATE INDEX CONCURRENTLY` を試してから本番に適用するフローが安全
- **インデックス肥大化の監視**:
  ```sql
  SELECT indexname, pg_size_pretty(pg_relation_size(indexname::regclass))
  FROM pg_indexes WHERE tablename = 'orders';
  ```
