# B-Treeインデックスの仕組みと計算量

## 概要

B-TreeはRDBMSが内部で使うインデックス構造の標準。PostgreSQL・MySQLどちらもデフォルトインデックスはB-Tree。
「なぜインデックスを貼るとSELECTが速くなるのか」「どんなクエリでは使われないのか」を知ることで、Neonのスロークエリを自分で診断・修正できる。
フルスタック開発でもN+1問題やページネーションのパフォーマンス改善は頻出テーマ。

---

## 仕組みの要点

### 木構造の概観

```
          [30 | 70]               ← ルートノード
         /    |    \
    [10|20] [40|60] [80|90]       ← 内部ノード
    /  |  \   ...
[葉] [葉] [葉]                    ← 葉ノード（実データへのポインタ）
```

- **ノード** = ディスクの1ページ（通常8KB）に収まる複数のキー
- **葉ノード** は双方向リンクリストで繋がっている → 範囲検索が高速
- **バランスを自動維持**：挿入/削除時に分割・マージを行い、全葉の深さが常に同じ

### 検索の流れ（`WHERE id = 42`）

1. ルートノードをロード（ディスクI/O × 1）
2. キーを比較して子ノードへ降下
3. 繰り返してO(log n)回で葉に到達
4. 葉ノードのポインタで実際の行データを取得

### 範囲検索の流れ（`WHERE age BETWEEN 20 AND 30`）

1. `age = 20` の葉ノードへ降下
2. 葉のリンクリストを右方向にたどりながら `age <= 30` の行を収集
3. フルスキャン不要でO(log n + k)（k = 結果件数）

---

## 計算量・パフォーマンス特性

| 操作 | 計算量 |
|------|--------|
| 検索 | O(log n) |
| 挿入 | O(log n) |
| 削除 | O(log n) |
| 範囲検索 | O(log n + k) |
| フルスキャン | O(n) ← インデックスなし |

- **nは行数ではなくノード数**。100万行でも木の深さは約4〜5段
- ディスクI/Oがボトルネック → メモリに乗るほど速い（shared_buffers）
- カーディナリティが低いカラム（boolean等）はインデックス効果が薄い

---

## コード例（PostgreSQL / SQL）

```sql
-- インデックス作成
CREATE INDEX idx_users_email ON users(email);

-- 複合インデックス（leftmost prefix ruleに注意）
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at);

-- 実行計画確認
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE user_id = 1 AND created_at > '2026-01-01';
-- "Index Scan using idx_orders_user_created" が出ればOK

-- インデックスが使われない例（関数適用）
SELECT * FROM users WHERE LOWER(email) = 'test@example.com';
-- 対策: 関数インデックス
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
```

---

## よくある誤解・落とし穴

- **「インデックスは多いほど良い」→ 誤り**：挿入/更新/削除のたびにインデックスも更新。書き込み負荷が増す
- **`LIKE '%keyword%'` はインデックス不使用**：前方一致（`LIKE 'key%'`）のみB-Tree有効。全文検索はGINインデックスを使う
- **複合インデックスの列順序が重要**：`(user_id, created_at)` はWHEREに`user_id`が含まれれば有効。`created_at`だけでは効かない
- **NULL値**：B-TreeはNULLを含む行もインデックスするが、`IS NULL`検索の動作はDBによって異なる
- **INSERTが遅い原因がインデックス**：大量バルクインサート時はインデックスを一時削除して後で再作成する手法もある

---

## 実務での使いどころ（FastAPI + Neon + Cloud Run）

- **Neon（PostgreSQL）**：`EXPLAIN ANALYZE` でSeq Scanが出たらインデックス追加を検討
- **FastAPIのエンドポイント**：ページネーション実装時に `OFFSET` が大きくなると遅化 → `cursor-based pagination` + インデックスで解決
- **Alembic マイグレーション**：`op.create_index()` でインデックスを管理。本番では `CONCURRENTLY` オプションでロックなし作成
- **N+1問題との違い**：インデックスはクエリ1回を速くする。N+1は発行回数の問題。両方意識する必要がある
- **Cloud Run（コールドスタート）**：接続プーリング（PgBouncer/Neon serverless driver）と組み合わせてレイテンシを最小化
