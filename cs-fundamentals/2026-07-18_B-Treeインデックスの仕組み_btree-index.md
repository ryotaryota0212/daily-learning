# B-Treeインデックスの仕組みと計算量

## 概要

B-Treeインデックスは、PostgreSQL・MySQL・SQLiteを含むほぼすべてのRDBMSのデフォルトインデックス構造。
なぜ「なんとなくインデックスを貼れば速くなる」ではなく「どのカラムにどう貼るか」を判断できるかが実力差になる。
Neonを使ったFastAPIバックエンドで、クエリが遅い原因を`EXPLAIN ANALYZE`で読み解く基礎になる。

---

## 仕組みの要点

### B-Tree の構造

```
          [30 | 70]                ← ルートノード
         /    |    \
   [10|20] [40|60] [80|90]        ← 内部ノード
   /  |  \   ...
[リーフ]  [リーフ]                 ← リーフノード（実データへのポインタ）
```

- **ノード** = ディスクページ（通常8KB）に詰まった複数のキー
- **リーフノード** = 実際の行データへのポインタ（ヒープファイルへの参照）を持つ
- **内部ノード** = 検索経路を絞るためのキーのみ（データポインタなし）
- リーフノードは双方向リンクリストで連結 → 範囲検索が効率的

### 検索の流れ（例：`WHERE id = 55`）

1. ルートノードをメモリに読み込む
2. `30 < 55 < 70` → 中央の子ノードへ
3. `40 < 55 < 60` → 該当する子ノードへ
4. リーフに到達 → 行の物理位置（ctid）を取得 → ヒープから行を読む

### 挿入・削除

- **挿入**：リーフに空きがあればそのまま追加。満杯なら**ノード分割**（スプリット）が発生
- **削除**：マーキングのみ（即時再構成はしない）→ `VACUUM` でページを整理
- 分割はルートまで伝播することがある → ツリーの高さが増える

---

## 計算量・パフォーマンス特性

| 操作 | 計算量 | 備考 |
|------|--------|------|
| 検索 | O(log N) | ツリーの高さ = log_m(N)（mは分岐数） |
| 挿入 | O(log N) | スプリット時もO(log N)で安定 |
| 削除 | O(log N) | |
| 範囲検索 | O(log N + K) | Kは取得件数 |

- 100万行のテーブル：高さは約3〜4レベル → **3〜4回のディスクI/O**で到達
- 分岐数（m）が大きいほどツリーは低く保たれる → B-Treeはディスク向けに設計

---

## コード例（PostgreSQL / SQL）

```sql
-- インデックス作成
CREATE INDEX idx_users_email ON users(email);

-- 複合インデックス（順序が重要）
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at);

-- 実行計画の確認
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE user_id = 42 AND created_at > '2026-01-01';

-- インデックスの使用状況を確認
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
WHERE tablename = 'orders';
```

**複合インデックスのポイント**：`(user_id, created_at)` は `WHERE user_id = ?` にも使えるが、
`WHERE created_at > ?` だけでは使えない（左端のカラムから順に有効）。

---

## よくある誤解・落とし穴

- **「インデックスは多いほど良い」は誤り**
  → 挿入・更新・削除のたびにインデックスも更新 → 書き込みが重くなる
- **低カーディナリティへのインデックスは逆効果になることがある**
  → `status` (active/inactive の2値) にインデックスを貼っても、全行の50%を返すクエリでは
    シーケンシャルスキャンの方が速い場合がある
- **LIKE '%keyword%' はインデックスを使えない**
  → 前方一致 `LIKE 'key%'` なら使える。全文検索は別途 GINインデックスを使う
- **NULLはB-Treeに含まれる**（PostgreSQLの場合）
  → `WHERE col IS NULL` にもインデックスが使える（MySQLとは異なる）
- **インデックス作成後に統計情報が古いと最適化されない**
  → `ANALYZE テーブル名` で統計を更新

---

## 実務での使いどころ（FastAPI + Neon + Cloud Run）

**Neon（PostgreSQL）での具体的な活用**

- **外部キーには必ずインデックスを貼る**：JOINで毎回スキャンされるのを防ぐ
  ```sql
  -- FastAPIのモデルでリレーションを貼ったら対応するインデックスも追加
  CREATE INDEX idx_posts_user_id ON posts(user_id);
  ```

- **`created_at` での絞り込みが多いなら複合インデックス**
  ```sql
  -- ユーザーごとの最新投稿を取るパターンに効く
  CREATE INDEX idx_posts_user_created ON posts(user_id, created_at DESC);
  ```

- **`EXPLAIN ANALYZE` を開発中に習慣化**：Cloud RunのコールドスタートとNeonの接続レイテンシを
  切り分けるためにも、クエリ自体のコストを把握しておく

- **Neonのブランチ機能でインデックス変更を検証**：本番DBのブランチを作り、
  インデックス追加前後で`EXPLAIN ANALYZE`の差を確認してからマイグレーションを流す
